# Mastering

In this tutorial, we will deploy and run the SD-Core 5G core network following Control and User Plane Separation (CUPS) principles. The radio and cell phone simulator will also be deployed on an isolated cluster. [Multipass](https://multipass.run/) is used to create separate VMs connected with [LXD](https://ubuntu.com/lxd) networking.

## 1. Prepare the Host machine

A machine running Ubuntu 22.04 with the following resources:

- At least one NIC with internet access
- 8 cores
- 24 GB RAM
- 150 GiB disk

### Networks

The following IP networks will be used to connect and isolate the network functions.

| Name         | Subnet        | Gateway IP |
| ------------ | ------------- | ---------- |
| `management` | 10.201.0.0/24 | 10.201.0.1 |
| `access`     | 10.202.0.0/24 | 10.202.0.1 |
| `core`       | 10.203.0.0/24 | 10.203.0.1 |
| `ran`        | 10.204.0.0/24 | 10.204.0.1 |

On the host machine, create local network bridges to be used by LXD by adding below configuration under `/etc/netplan/99-sdcore-networks.yaml`.
Before creating the configuration of the network bridges, please make sure that:

- mgmt-br route metric value is higher than your default route's metric
- core-br metric value is higher than your mgmt-br route's metric

Change the metrics of SD-Core routes which are indicated with comments below, relatively to your default route's metric if required.

```console
cat << EOF | sudo tee /etc/netplan/99-sdcore-networks.yaml
# /etc/netplan/99-sdcore-networks.yaml
network:
  bridges:
    mgmt-br:
      addresses:
        - 10.201.0.14/24
      routes:
        - to: default
          via: 10.201.0.1
          metric: 110 # Set the value higher than your default route's metric
    access-br:
      addresses:
        - 10.202.0.14/24
      routes:
        - to: 10.204.0.0/24
          via: 10.202.0.1
    core-br:
      addresses:
        - 10.203.0.14/24
      routes:
        - to: default
          via: 10.203.0.1
          metric: 203 # Set the value higher than your mgmt-br route's metric
    ran-br:
      addresses:
        - 10.204.0.14/24
      routes:
        - to: 10.202.0.0/24
          via: 10.204.0.1

  version: 2
EOF
```

Arrange the file permissions and apply the network configuration.

```console
sudo chmod 600 /etc/netplan/99-sdcore-networks.yaml
sudo netplan apply
```

```{note}
Applying new netplan configuration may produce warnings related to file permissions being too open. You may safely disregard them.
```

### Install and Configure LXD

Install LXD:

```console
sudo snap install lxd
```

Create custom storage pool and LXD project:

```console
lxc storage create sdcore zfs size=150GiB
lxc project create sdcore --config features.images=true --config features.networks=true --config features.networks.zones=true --config features.profiles=true --config features.storage.buckets=true --config features.storage.volumes=true
```

```{note}
When using LXD version lower than 5.9, the command above will produce the following error:
> Error: Invalid project configuration key "features.networks.zones"

Please upgrade your LXD to the latest version or remove unsupported config from the command.
```

Create the configuration to initialize the LXD.

```console
cat << EOF | sudo tee ~/preseed.yaml
config: {}
networks: []
storage_pools:
- config:
    size: 150GiB
    source: /var/snap/lxd/common/lxd/disks/sdcore.img
    zfs.pool_name: sdcore
  description: ""
  name: sdcore
  driver: zfs
profiles:
- config: {}
  description: Default LXD profile
  devices:
    root:
      path: /
      pool: sdcore
      type: disk
  name: default
projects:
- config:
    features.images: "true"
    features.networks: "true"
    features.networks.zones: "true" # Omit if the LXD version is lower than 5.9.
    features.profiles: "true"
    features.storage.buckets: "true"
    features.storage.volumes: "true"
  description: SD-Core project
  name: sdcore
EOF
```

```console
lxd init --preseed < ~/preseed.yaml
```

### Install and configure Multipass

Install Multipass and set LXD as local driver:

```console
sudo snap install multipass
multipass set local.driver=lxd
```

Wait a few seconds if you get the output: `set failed: cannot connect to the multipass socket` and retry to set local driver again.

Connect Multipass to LXD.

```console
sudo snap connect multipass:lxd lxd
```

## 2. Create Virtual Machines

To complete this tutorial, you will need seven virtual machines with access to the networks as follows.

| Machine                              | CPUs | RAM | Disk | Networks                       |
| ------------------------------------ | ---- | --- | ---- | ------------------------------ |
| DNS Server                           | 1    | 1g  | 10g  | `management`                   |
| Control Plane Kubernetes Cluster     | 4    | 8g  | 40g  | `management`                   |
| User Plane Kubernetes Cluster        | 2    | 4g  | 20g  | `management`, `access`, `core` |
| Juju Controller + Kubernetes Cluster | 4    | 6g  | 40g  | `management`                   |
| gNB Simulator Kubernetes Cluster     | 2    | 3g  | 20g  | `management`, `ran`            |
| RAN Access Router                    | 1    | 1g  | 10g  | `management`, `ran` , `access` |
| Core Router                          | 1    | 1g  | 10g  | `management`, `core`           |

Create VMs with Multipass:

```console
multipass launch -c 1 -m 1G -d 10G -n dns --network mgmt-br jammy
multipass launch -c 4 -m 8G -d 40G -n control-plane --network mgmt-br jammy
multipass launch -c 2 -m 4G -d 20G -n user-plane  --network mgmt-br --network core-br --network access-br jammy
multipass launch -c 4 -m 6G -d 40G -n juju-controller --network mgmt-br jammy
multipass launch -c 2 -m 3G -d 20G -n gnbsim --network mgmt-br --network ran-br jammy
multipass launch -c 1 -m 1G -d 10G -n ran-access-router --network mgmt-br --network ran-br --network access-br jammy
multipass launch -c 1 -m 1G -d 10G -n core-router --network mgmt-br --network core-br jammy
```

Wait until all VMs are in a `Running` state.

### Checkpoint 1: Are the VM's ready ?

You should be able to see all the VMs in a `Running` state with their default IP addresses by executing the following command:

```console
multipass list
```

The output should be similar to the following

```
Name                    State             IPv4             Image
juju-controller         Running           10.231.204.5     Ubuntu 22.04 LTS
core-router             Running           10.231.204.200   Ubuntu 22.04 LTS
control-plane           Running           10.231.204.202   Ubuntu 22.04 LTS
dns                     Running           10.231.204.96    Ubuntu 22.04 LTS
gnbsim                  Running           10.231.204.24    Ubuntu 22.04 LTS
ran-access-router       Running           10.231.204.220   Ubuntu 22.04 LTS
user-plane              Running           10.231.204.121   Ubuntu 22.04 LTS
```

### Install the DNS Server

Log in to the `dns` VM.

```console
multipass shell dns
```

First, replace the content of `/etc/netplan/50-cloud-init.yaml` as following to configure `mgmt` interface IP address as `10.201.0.100`.

```console
cat << EOF | sudo tee /etc/netplan/50-cloud-init.yaml
network:
    ethernets:
        enp5s0:
            dhcp4: true
        enp6s0:
            dhcp4: false
            addresses:
              - 10.201.0.100/24
    version: 2
EOF
```

Apply the network configuration.

```console
sudo chmod 600 /etc/netplan/50-cloud-init.yaml
sudo netplan apply
```

Install the DNS server:

```console
sudo apt update
sudo apt install dnsmasq -y
sudo systemctl disable systemd-resolved
sudo systemctl stop systemd-resolved
sudo systemctl restart dnsmasq
```

Configure dnsmasq:

```console
cat << EOF | sudo tee /etc/dnsmasq.conf
no-resolv
server=8.8.8.8
server=8.8.4.4
domain=mgmt
addn-hosts=/etc/hosts.tutorial
EOF
```

Update resolv.conf as we are no longer using systemd-resolved:
```console
sudo rm /etc/resolv.conf
echo 127.0.0.1 | sudo tee /etc/resolv.conf
```

The following IP addresses are used in this tutorial and must be present in the DNS Server that all hosts are using.

| Name                                   | IP Address   | Purpose                                                  |
| -------------------------------------- | ------------ | -------------------------------------------------------- |
| `juju-controller.mgmt`                 | 10.201.0.104 | Management address for Juju machine                      |
| `control-plane.mgmt`                   | 10.201.0.101 | Management address for control plane cluster machine     |
| `user-plane.mgmt`                      | 10.201.0.102 | Management address for user plane cluster machine        |
| `gnbsim.mgmt`                          | 10.201.0.103 | Management address for the gNB Simulator cluster machine |
| `api.juju-controller.mgmt`             | 10.201.0.50  | Juju controller address                                  |
| `cos.mgmt`                             | 10.201.0.51  | Canonical Observability Stack address                    |
| `amf.mgmt`                             | 10.201.0.52  | Externally reachable control plane endpoint for the AMF  |
| `control-plane-nms.control-plane.mgmt` | 10.201.0.53  | Externally reachable control plane endpoint for the NMS  |
| `upf.mgmt`                             | 10.201.0.200 | Externally reachable control plane endpoint for the UPF  |

Add records under /etc/hosts:

```console
cat << EOF | sudo tee -a /etc/hosts.tutorial
10.201.0.50    api.juju-controller.mgmt
10.201.0.51    cos.mgmt
10.201.0.52    amf.mgmt
10.201.0.53    control-plane-nms.control-plane.mgmt
10.201.0.101   control-plane.mgmt
10.201.0.102   user-plane.mgmt
10.201.0.103   gnbsim.mgmt
10.201.0.104   juju-controller.mgmt
10.201.0.200   upf.mgmt
EOF
```

Reload the DNS configuration:

```console
sudo systemctl restart dnsmasq
```

### Checkpoint 2: Is the DNS server running properly?

You should expect to see that `dnsmasq` service is up and running.

```console
$ sudo systemctl status dnsmasq
dnsmasq.service - dnsmasq - A lightweight DHCP and caching DNS server
     Loaded: loaded (/lib/systemd/system/dnsmasq.service; enabled; vendor preset: enabled)
     Active: active (running) since Thu 2024-01-11 13:46:34 +03; 6ms ago
    Process: 2611 ExecStartPre=/etc/init.d/dnsmasq checkconfig (code=exited, status=0/SUCCESS)
    Process: 2619 ExecStart=/etc/init.d/dnsmasq systemd-exec (code=exited, status=0/SUCCESS)
    Process: 2628 ExecStartPost=/etc/init.d/dnsmasq systemd-start-resolvconf (code=exited, status=0/SUCCESS)
```

Test the DNS resolution and expect that the Nameserver is able to resolve the registered host names.

```console
host upf.mgmt
```

You should see `user-plane.mgmt has address 10.201.0.102`

Log out of the VM.

### Add DNS server and routes to the other VM's

#### User-plane VM

Log in to the `user-plane` VM.

```console
multipass shell user-plane
```

Configure IP address for `mgmt`, `core` and `access` interfaces, add nameservers for the `mgmt` interface and add route from `access` to `ran` network by replacing the content of `/etc/netplan/50-cloud-init.yaml` as following:

```console
cat << EOF | sudo tee /etc/netplan/50-cloud-init.yaml
network:
    ethernets:
        enp5s0:
            dhcp4: true
        enp6s0:
            dhcp4: false
            addresses:
              - 10.201.0.102/24
            nameservers:
                search: [mgmt]
                addresses: [10.201.0.100]
            optional: true
        enp7s0:
            dhcp4: false
            addresses:
              - 10.203.0.100/24
            optional: true
        enp8s0:
            dhcp4: false
            addresses:
              - 10.202.0.100/24
            routes:
              - to: 10.204.0.0/24
                via: 10.202.0.1
            optional: true
    version: 2
EOF
```

Apply the network configuration.

```console
sudo chmod 600 /etc/netplan/50-cloud-init.yaml
sudo netplan apply
```

Check the current DNS server.

```console
$ resolvectl
Link 3 (enp6s0)
    Current Scopes: DNS
         Protocols: +DefaultRoute +LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
Current DNS Server: 10.201.0.100
       DNS Servers: 10.201.0.100
        DNS Domain: mgmt
```

Check the route from `access` interface to the `ran` network.

```console
ip route
```

You should see the following routes in addition to the regular host routes.

```console
10.201.0.0/24 dev enp6s0 proto kernel scope link src 10.201.0.102
10.202.0.0/24 dev enp8s0 proto kernel scope link src 10.202.0.100
10.203.0.0/24 dev enp7s0 proto kernel scope link src 10.203.0.100
10.204.0.0/24 via 10.202.0.1 dev enp8s0 proto static
```

Log out of the VM.

#### Control-plane VM

Log in to the `control-plane` VM.

```console
multipass shell control-plane
```

Configure IP address and nameservers for `mgmt` interface by replacing the content of `/etc/netplan/50-cloud-init.yaml` as following:

```console
cat << EOF | sudo tee /etc/netplan/50-cloud-init.yaml
network:
    ethernets:
        enp5s0:
            dhcp4: true
        enp6s0:
            dhcp4: false
            addresses:
              - 10.201.0.101/24
            nameservers:
                search: [mgmt]
                addresses: [10.201.0.100]
            optional: true
    version: 2
EOF
```

Apply the network configuration.

```console
sudo chmod 600 /etc/netplan/50-cloud-init.yaml
sudo netplan apply
```

Check the current DNS server.

```console
resolvectl
```

Log out of the VM.

#### Gnbsim VM

Log in to the `gnbsim` VM.

```console
multipass shell gnbsim
```

Configure IP address for `mgmt` and `ran` interfaces add nameservers for the `mgmt` interface and add route from `ran` to `access` network by replacing the content of `/etc/netplan/50-cloud-init.yaml` as following:

```console
cat << EOF | sudo tee /etc/netplan/50-cloud-init.yaml
network:
    ethernets:
        enp5s0:
            dhcp4: true
        enp6s0:
            dhcp4: false
            addresses:
              - 10.201.0.103/24
            nameservers:
                search: [mgmt]
                addresses: [10.201.0.100]
            optional: true
        enp7s0:
            dhcp4: false
            addresses:
              - 10.204.0.100/24
            routes:
              - to: 10.202.0.0/24
                via: 10.204.0.1
            optional: true
    version: 2
EOF
```

Apply the network configuration.

```console
sudo chmod 600 /etc/netplan/50-cloud-init.yaml
sudo netplan apply
```

Check the current DNS server.

```console
resolvectl
```

Check the route from `ran` interface to the `access` network.

```console
ip route
```

You should see the following routes in addition to the regular host routes.

```console
10.201.0.0/24 dev enp6s0 proto kernel scope link src 10.201.0.103
10.202.0.0/24 via 10.204.0.1 dev enp7s0 proto static
10.204.0.0/24 dev enp7s0 proto kernel scope link src 10.204.0.100
```

```console
$ ip route
...
10.202.0.0/24 via 10.204.0.1 dev enp7s0 proto static
```

Log out of the VM.

#### Juju-controller VM

Log in to the `juju-controller` VM.

```console
multipass shell juju-controller
```

Configure IP address and nameservers for `mgmt` interface by replacing the content of `/etc/netplan/50-cloud-init.yaml` as following:

```console
cat << EOF | sudo tee /etc/netplan/50-cloud-init.yaml
network:
    ethernets:
        enp5s0:
            dhcp4: true
        enp6s0:
            dhcp4: false
            addresses:
              - 10.201.0.104/24
            nameservers:
                search: [mgmt]
                addresses: [10.201.0.100]
            optional: true
    version: 2
EOF
```

Apply the network configuration.

```console
sudo chmod 600 /etc/netplan/50-cloud-init.yaml
sudo netplan apply
```

Check the current DNS server.

```console
resolvectl
```

Log out of the VM.

#### RAN-access-router VM

Log in to the `ran-access-router` VM.

```console
multipass shell ran-access-router
```

Configure IP address for `mgmt`, `ran` and `access` interfaces by replacing the content of `/etc/netplan/50-cloud-init.yaml` as following:

```console
cat << EOF | sudo tee /etc/netplan/50-cloud-init.yaml
network:
    ethernets:
        enp5s0:
            dhcp4: true
        enp6s0:
            dhcp4: false
            addresses:
              - 10.201.0.110/24
            optional: true
        enp7s0:
            dhcp4: false
            addresses:
              - 10.204.0.1/24
            optional: true
        enp8s0:
            dhcp4: false
            addresses:
              - 10.202.0.1/24
            optional: true
    version: 2
EOF
```

Apply the network configuration.

```console
sudo chmod 600 /etc/netplan/50-cloud-init.yaml
sudo netplan apply
```

The `access-gateway-ip` is expected to forward the packets from the `access-interface` to the `gnb-subnet`.

Set up IP forwarding

```console
echo net.ipv4.ip_forward=1 | sudo tee /etc/sysctl.conf
sudo sysctl -w net.ipv4.ip_forward=1
```

Log out of the VM.

#### Core-router VM

Log in to the `core-router` VM.

```console
multipass shell core-router
```

Configure IP address for `mgmt` and `core` interfaces by replacing the content of `/etc/netplan/50-cloud-init.yaml` as following:

```console
cat << EOF | sudo tee /etc/netplan/50-cloud-init.yaml
network:
    ethernets:
        enp5s0:
            dhcp4: true
        enp6s0:
            dhcp4: false
            addresses:
              - 10.201.0.114/24
            optional: true
        enp7s0:
            dhcp4: false
            addresses:
              - 10.203.0.1/24
            optional: true
    version: 2
EOF
```

Apply the network configuration.

```console
sudo chmod 600 /etc/netplan/50-cloud-init.yaml
sudo netplan apply
```

Set up IP forwarding and NAT:

```console
cat << EOF | sudo tee /etc/rc.local
#!/bin/bash
iptables -t nat -A POSTROUTING -o enp5s0 -j MASQUERADE -s 10.203.0.0/24
EOF

sudo chmod +x /etc/rc.local
sudo /etc/rc.local
sudo sysctl -w net.ipv4.ip_forward=1 | sudo tee -a /etc/sysctl.conf
```

Log out of the VM.

## 3. Configure VMs for SD-Core Deployment

This section covers setting up the SSH keys and installation of Juju and MicroK8s on VMs which are going to build up the infrastructure for SD-Core.

As we are going to be copying files around using ssh, we now will set up a new ssh key on the host running the tutorial:

```console
ssh-keygen -f ~/tutorial_rsa -N ""
```

Copy the keys to all the SD-Core VMs:

```console
for VM in control-plane juju-controller gnbsim user-plane
do
  multipass transfer ~/tutorial_rsa ${VM}:.ssh/id_rsa
  multipass transfer ~/tutorial_rsa.pub ${VM}:.ssh/id_rsa.pub
  multipass exec ${VM} -- sh -c 'cat .ssh/id_rsa.pub >> .ssh/authorized_keys'
done
```

```{note}
You may now delete the `tutorial_rsa` and `tutorial_rsa.pub` files from the host.
```

Log out of the VM.

### Prepare SD-Core Control Plane VM

Login to the `control-plane` VM.

```console
multipass shell control-plane
```

Install Microk8s.

```console
sudo snap install microk8s --channel=1.27-strict/stable
sudo microk8s enable hostpath-storage
sudo usermod -a -G snap_microk8s $USER
```

The control plane needs to expose two services: the AMF and the NMS. In this step, we enable the MetalLB add on in Microk8s, and give it a range of two IP addresses:

```console
sudo microk8s enable metallb:10.201.0.52-10.201.0.53
```

Now update MicroK8s DNS to point to our DNS server

```console
sudo microk8s disable dns
sudo microk8s enable dns:10.201.0.100
```

Export the Kubernetes configuration and copy that to the `juju-controller` VM:

```console
sudo microk8s.config > control-plane-cluster.yaml
scp control-plane-cluster.yaml juju-controller.mgmt:
```

### Prepare SD-Core User Plane VM

Log in to the `user-plane` VM.

```console
multipass shell user-plane
```

Install MicroK8s, configure MetalLB to expose 1 IP address for the UPF (`10.201.0.200`) and enable the Multus plugin:

```console
sudo snap install microk8s --channel=1.27-strict/stable
sudo microk8s enable hostpath-storage
sudo microk8s enable metallb:10.201.0.200-10.201.0.200
sudo microk8s addons repo add community \
    https://github.com/canonical/microk8s-community-addons \
    --reference feat/strict-fix-multus
sudo microk8s enable multus
sudo usermod -a -G snap_microk8s $USER
```

Update MicroK8s DNS to point to our DNS server

```console
sudo microk8s disable dns
sudo microk8s enable dns:10.201.0.100
```

Export the Kubernetes configuration and copy that to the `juju-controller` VM:

```console
sudo microk8s.config > user-plane-cluster.yaml
scp user-plane-cluster.yaml juju-controller.mgmt:
```


In this guide, the following network interfaces are available on the SD-Core `user-plane` VM:

| Interface Name | Purpose                                                                                                                                                           |
| -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| enp6s0         | internal Kubernetes management interface. This maps to the `management` subnet.                                                                                   |
| enp7s0         | core interface. This maps to the `core` subnet.                                                                                                                   |
| enp8s0         | access interface. This maps to the `access` subnet. Note that internet egress is required here and routing tables are already set to route gNB generated traffic. |

Now we create the MACVLAN bridges for `enp7s0` and `enp8s0`. These instructions are put into a file that is executed on reboot so the interfaces will come back.

```console
cat << EOF | sudo tee /etc/rc.local
#!/bin/bash

sudo ip link add access link enp8s0 type macvlan mode bridge
sudo ip link set dev access up
sudo ip link add core link enp7s0 type macvlan mode bridge
sudo ip link set dev core up
EOF
sudo chmod +x /etc/rc.local
sudo /etc/rc.local
```

Log out of the VM.

### Prepare gNB Simulator VM

Log in to the `gnbsim` VM.

```console
multipass shell gnbsim
```

Install MicroK8s and add the Multus plugin:

```console
sudo snap install microk8s --channel=1.27-strict/stable
sudo microk8s enable hostpath-storage
sudo microk8s addons repo add community \
    https://github.com/canonical/microk8s-community-addons \
    --reference feat/strict-fix-multus
sudo microk8s enable multus
sudo usermod -a -G snap_microk8s $USER
```

Update MicroK8s DNS to point to our DNS server

```console
sudo microk8s disable dns
sudo microk8s enable dns:10.201.0.100
```

Export the Kubernetes configuration and copy that to the `juju-controller` VM:

```console
sudo microk8s.config > gnb-cluster.yaml
scp gnb-cluster.yaml juju-controller.mgmt:
```

In this guide, the following network interfaces are available on the `gnbsim` VM:

| Interface Name | Purpose                                                                         |
| -------------- | ------------------------------------------------------------------------------- |
| enp6s0         | internal Kubernetes management interface. This maps to the `management` subnet. |
| enp7s0         | ran interface. This maps to the `ran` subnet.                                   |

Now we create the MACVLAN bridges for `enp7s0`, and label them accordingly:

```console
cat << EOF | sudo tee /etc/rc.local
#!/bin/bash

sudo ip link add ran link enp7s0 type macvlan mode bridge
sudo ip link set dev ran up
EOF
sudo chmod +x /etc/rc.local
sudo /etc/rc.local
```

Log out of the VM.

### Prepare the Juju Controller VM

Log in to the `juju-controller` VM.

```console
multipass shell juju-controller
```

Begin by installing MicroK8s to hold the Juju controller. Configure MetalLB to expose one IP address for the controller (`10.201.0.50`) and one for the Canonical Observability Stack (`10.201.0.51)`.

```console
sudo snap install microk8s --channel=1.27-strict/stable
sudo microk8s enable hostpath-storage
sudo microk8s enable metallb:10.201.0.50-10.201.0.51
sudo usermod -a -G snap_microk8s $USER
newgrp snap_microk8s
```

Update MicroK8s DNS to point to our DNS server

```console
sudo microk8s disable dns
sudo microk8s enable dns:10.201.0.100
```

Install Juju and bootstrap the controller to the local Microk8s install as a load balancer service. This will expose the Juju controller on the first allocated MetalLB address.

```console
mkdir -p ~/.local/share/juju
sudo snap install juju --channel=3.4/stable
juju bootstrap microk8s --config controller-service-type=loadbalancer sdcore
```

At this point, the Juju controller is ready to start managing external clouds. Add the Kubernetes clusters representing the user plane, control plane, and gNB simulator to Juju. This is done by using the Kubernetes configuration file generated when setting up the clusters above.

```console
export KUBECONFIG=control-plane-cluster.yaml
juju add-k8s control-plane-cluster --controller sdcore
export KUBECONFIG=user-plane-cluster.yaml
juju add-k8s user-plane-cluster --controller sdcore
export KUBECONFIG=gnb-cluster.yaml
juju add-k8s gnb-cluster --controller sdcore
```

Log out of the VM.

```{note}
Due to the newgrp command you will need to log out twice as it started a new shell.
```

## 4. Deploy SD-Core Control Plane

The following steps build on the Juju controller which was bootstrapped and knows how to manage the SD-Core Control Plane Kubernetes cluster.

First, create a Juju overlay file that specifies the Access and Mobility Management Function (AMF) host name and IP address for sharing with the radios. This host name must be resolvable by the gNB and the IP address must be reachable and resolve to the AMF unit. In the bootstrap step, we set the Control Plane MetalLB range to start at `10.201.0.52`, so that is what we use in the configuration.

Log into the `juju-controller` VM.

```console
multipass shell juju-controller
```

Create the Juju overlay file where we specify the IP address and hostname for the AMF.

```console
cat << EOF > control-plane-overlay.yaml
applications:
  amf:
    options:
      external-amf-ip: 10.201.0.52
      external-amf-hostname: amf.mgmt
EOF
```

Next, we create a Juju model to represent the Control Plane, using the cloud `control-plane-cluster`, which was created in the Bootstrap step.

```console
juju add-model control-plane control-plane-cluster
```

Deploy the control plane Juju bundle:

```console
juju deploy sdcore-control-plane-k8s --trust --channel=beta --overlay control-plane-overlay.yaml
```

Expose the Software as a Service offer for the AMF.

```console
juju offer control-plane.amf:fiveg-n2
```

### Checkpoint 3: Does AMF External load balancer service exist ?

You should be able to see the AMF External load balancer service in Kubernetes. Log in to the `control-plane` VM and execute the following command:

```console
multipass shell control-plane
microk8s.kubectl get services -A | grep LoadBalancer
```

This will show output similar to the following, indicating:

- The AMF is exposed on 10.201.0.52 SCTP port 38412
- The NMS is exposed on 10.201.0.53 TCP ports 80 and 443

These IP addresses came from MetalLB and were configured in the bootstrap step.

```console
control-plane    amf-external  LoadBalancer  10.152.183.179  10.201.0.52   38412:30408/SCTP
control-plane    traefik-k8s   LoadBalancer  10.152.183.28   10.201.0.53   80:32349/TCP,443:31925/TCP
```

## 5. Deploy SD-Core User Plane

On the `juju-controller` VM, create an overlay file for the UPF deployment. The following parameters can be set:

- `access-gateway-ip`: this is the IP address of the gateway that knows how to route traffic from the UPF towards the gNB subnet
- `access-interface`: the name of the MACVLAN interface on the Kubernetes host cluster to bridge to the `access` subnet
- `access-ip`: the IP address for the UPF to use on the `access` subnet
- `core-gateway-ip`: this is the IP address of the gateway that knows how to route traffic from the UPF towards the internet
- `core-interface`: the name of the MACVLAN interface on the Kubernetes host cluster to bridge to the `core` subnet
- `core-ip`: the IP address for the UPF to use on the `core` subnet
- `external-upf-hostname`: the DNS name of the UPF
- `gnb-subnet`: the subnet CIDR where the gNB radios are reachable.

Create the UPF overlay file on the `juju-controller` VM.

```console
cat << EOF > upf-overlay.yaml
applications:
  upf:
    options:
      access-gateway-ip: 10.202.0.1
      access-interface: access
      access-ip: 10.202.0.10/24
      core-gateway-ip: 10.203.0.1
      core-interface: core
      core-ip: 10.203.0.10/24
      external-upf-hostname: upf.mgmt
      gnb-subnet: 10.204.0.0/24
EOF
```

Next, we create a Juju model to represent the User Plane, using the cloud `user-plane-cluster`, which was created in the Bootstrap step.

```console
juju add-model user-plane user-plane-cluster
```

Deploy user plane bundle:

```console
juju deploy sdcore-user-plane-k8s --trust --channel=beta --overlay upf-overlay.yaml
```

Now expose the UPF service offering with Juju.

```console
juju offer user-plane.upf:fiveg_n4
```

### Checkpoint 4: Does UPF External load balancer service exist ?

You should be able to see the UPF External load balancer service in Kubernetes. Log in to the `user-plane` VM and display the load balancer service:

```console
multipass shell user-plane
microk8s.kubectl get services -A | grep LoadBalancer
```

This should produce output similar to the following indicating that the PFCP agent of the UPF is exposed on `10.201.0.200` UDP port 8805

```console
user-plane  upf-external  LoadBalancer  10.152.183.126  10.201.0.200  8805:31101/UDP
```

## 6. Deploy the gNB Simulator

Create a new Juju model named `gnbsim` on the `juju-controller` VM:

```console
juju add-model gnbsim gnb-cluster
```

Deploy the simulator to the `gnbsim-cluster`. The simulator needs to know the following:

- `gnb-interface`: the name of the MACVLAN interface to use on the host
- `gnb-ip-address`: the IP address to use on the gnb interface
- `icmp-packet-destination`: the target IP address to ping. If there is no egress to the internet on your core network, any IP that is reachable from the UPF should work.
- `upf-gateway`: the IP address of the gateway between the RAN and Access networks
- `upf-subnet`: subnet where the UPFs are located (also called Access network)

```console
juju deploy sdcore-gnbsim-k8s gnbsim --channel=beta \
--config gnb-interface=ran \
--config gnb-ip-address=10.204.0.10/24 \
--config icmp-packet-destination=8.8.8.8 \
--config upf-gateway=10.204.0.1 \
--config upf-subnet=10.202.0.0/24
```

Now we use the integration between the gnbsim operator and the AMF operator to exchange the AMF location information.

```console
juju consume control-plane.amf
juju integrate gnbsim:fiveg-n2 amf:fiveg-n2
```

And present the offering of the gNB identity for integrating with the NMS.

```console
juju offer gnbsim.gnbsim:fiveg_gnb_identity
```

## 7. Configure SD-Core

Still on the `juju-controller` VM, switch to the `control-plane` model.

```console
juju switch control-plane
```

Relate the UPF to the NMS for exchanging identity information.

```console
juju consume user-plane.upf
juju integrate upf:fiveg_n4 nms:fiveg_n4
juju consume gnbsim.gnbsim
juju integrate gnbsim:fiveg_gnb_identity nms:fiveg_gnb_identity
```

Configure Traefik to use an external hostname:

```console
juju config traefik-k8s external_hostname=10.201.0.53.nip.io
```

Here, replace `10.201.0.53` with the Application IP address of the `traefik-k8s` application. You can find it by running `juju status traefik-k8s`.

Retrieve the NMS address:

```console
juju run traefik-k8s/0 show-proxied-endpoints
```

The output should be `http://control-plane-nms.10.201.0.53.nip.io/`. Navigate to this address in your browser.

In the Network Management System (NMS), create a network slice with the following attributes:

- Name: `Tutorial`
- MCC: `208`
- MNC: `93`
- UPF: `upf.mgmt:8805`
- gNodeB: `gnbsim-gnbsim-gnbsim (tac:1)`

You should see the following network slice created. Note the device group has been expanded to show the default group that is created in the slice for you.

```{image} ../images/nms_tutorial_network_slice_with_device_group.png
:alt: NMS Network Slice
:align: center
```

We will now add a subscriber with the IMSI that was provided to the gNB simulator. Navigate to Subscribers and click on Create. Fill in the following:

- IMSI: `208930100007487`
- OPC: `981d464c7c52eb6e5036234984ad0bcf`
- Key: `5122250214c33e723a5dd523fc145fc0`
- Sequence Number: `16f3b3f70fc2`
- Network Slice: `Tutorial`
- Device Group: `Tutorial-default`

## 8. Integrate SD-Core with Observability

We will integrate the 5G core network with the Canonical Observability Stack (COS). All commands are to be executed on the `juju-controller` VM.

### Deploy the `cos-lite` bundle

Create a new Juju model named `cos` on the `juju-controller` cloud:

```console
juju add-model cos microk8s
```

Deploy the `cos-lite` charm bundle. As this is being deployed to the Juju controller's Kubernetes, it will use the second MetalLB IP address (`10.201.0.51`) as its external IP.

```console
juju deploy cos-lite --trust
```

You can validate the status of the deployment by running `juju status`. COS is ready when all the charms are in the `Active/Idle` state.

### Deploy the `cos-configuration-k8s` charm

Deploy the `cos-configuration-k8s` charm with the following SD-Core COS configuration:

```console
juju deploy cos-configuration-k8s \
  --config git_repo=https://github.com/canonical/sdcore-cos-configuration \
  --config git_branch=main \
  --config git_depth=1 \
  --config grafana_dashboards_path=grafana_dashboards/sdcore/
```

Integrate it to Grafana:

```console
juju integrate cos-configuration-k8s grafana
```

### Integrate Grafana Agent with Prometheus

First, offer the following integrations from Prometheus and Loki for use in other models on the `juju-controller` VM:

```console
juju offer cos.prometheus:receive-remote-write
juju offer cos.loki:logging
```

Then, consume the integrations from the `control-plane` model:

```console
juju switch control-plane
juju consume cos.prometheus
juju consume cos.loki
```

Integrate `grafana-agent-k8s` (in the `control-plane` model) with `prometheus` and `loki` (in the `cos` model):

```console
juju integrate prometheus:receive-remote-write grafana-agent-k8s:send-remote-write
juju integrate loki:logging grafana-agent-k8s:logging-consumer
```

Now, do the same for the `user-plane` model:

```console
juju switch user-plane
juju consume cos.prometheus
juju consume cos.loki
juju integrate prometheus:receive-remote-write grafana-agent-k8s:send-remote-write
juju integrate loki:logging grafana-agent-k8s:logging-consumer
```

### Login to Grafana

Retrieve the Grafana admin password on the `juju-controller` VM:

```console
juju switch cos
juju run grafana/leader get-admin-password
```

This produces output similar to the following:

```console
Running operation 1 with 1 task
  - task 2 on unit-grafana-0

Waiting for task 2...
admin-password: c72uEq8FyGRo
url: http://10.201.0.51/cos-grafana

```

## Checkpoint 5: Is Grafana dashboard available ?

In your browser, navigate to the URL from the output (`https://10.201.0.51/cos-grafana`). Login using the "admin" username and the admin password provided in the last command. Click on "Dashboards" -> "Browse" and select "5G Network Overview".

This dashboard presents an overview of your 5G Network status. Keep this page open, we will revisit it shortly.

```{image} ../images/grafana_5g_dashboard_sim_before.png
:alt: Initial Grafana dashboard showing UPF status
:align: center
```

## 9. Run the 5G simulation

On the `juju-controller` VM, switch to the `gnbsim` model.

```console
juju switch gnbsim
```

Start the simulation.

```console
juju run gnbsim/leader start-simulation
```

The simulation executed successfully if you see `success: "true"` as one of the
output messages:

```console
ubuntu@juju-controller:~$ juju run gnbsim/leader start-simulation
Running operation 1 with 1 task
  - task 2 on unit-gnbsim-0

Waiting for task 2...
info: run juju debug-log to get more information.
success: "true"
```

## Checkpoint 6: Check the simulation logs to see the communication between elements and the data exchange

### gNB Simulation Logs

Let's take a look at the juju debug-log now by running the following command:

```console
juju debug-log --no-tail
```

This will emit the full log of the simulation starting with the following message:

```console
unit-gnbsim-0: 16:43:50 INFO unit.gnbsim/0.juju-log gnbsim simulation output:
```

As there is a lot of output, we can better understand if we filter by specific elements. For example, let's take a look at the control plane transport of the log. To do that, we search for `ControlPlaneTransport` in the Juju debug-log. This shows the simulator locating the AMF and exchanging data with it.

```console
$ juju debug-log | grep ControlPlaneTransport
2023-11-30T16:43:40Z [TRAC][GNBSIM][GNodeB][ControlPlaneTransport] Connecting to AMF
2023-11-30T16:43:40Z [INFO][GNBSIM][GNodeB][ControlPlaneTransport] Connected to AMF, AMF IP: 10.201.0.52 AMF Port: 38412
...
```

We can do the same for the user plane transport to see it starts on the RAN network with IP address `10.204.0.10` as we requested, and it is communicating with our UPF at `10.202.0.10` as expected.

To follow the UE itself, we can filter by the IMSI.

```console
juju debug-log | grep imsi-208930100007487
```

### Control Plane Logs

You may view the control plane logs by logging into the control plane cluster and using Kubernetes commands as follows:

```bash
microk8s.kubectl logs -n control-plane -c amf amf-0 --tail 70
microk8s.kubectl logs -n control-plane -c ausf ausf-0 --tail 70
microk8s.kubectl logs -n control-plane -c nrf nrf-0 --tail 70
microk8s.kubectl logs -n control-plane -c nssf nssf-0 --tail 70
microk8s.kubectl logs -n control-plane -c pcf pcf-0 --tail 70
microk8s.kubectl logs -n control-plane -c smf smf-0 --tail 70
microk8s.kubectl logs -n control-plane -c udm udm-0 --tail 70
microk8s.kubectl logs -n control-plane -c udr udr-0 --tail 70
```

## Checkpoint 7: View the metrics

### Grafana Metrics

You can also revisit the Grafana dashboard to view the metrics for the test run. You can see the IMSI is connected and has received an IP address. There is now one active PDU session, and the ping test throughput can be seen in the graphs.

```{image} ../images/grafana_5g_dashboard_sim_after.png
:alt: Grafana dashboard showing throughput metrics
:align: center
```

## 10. Review

We have deployed 4 Kubernetes clusters, bootstrapped a Juju controller to manage them all, and deployed portions of the Charmed 5g SD-Core software according to CUPS principles. You know have 5 Juju models as follows

- `control-plane` where all the control functions are deployed
- `controller` where Juju manages state of the models
- `cos` where the Canonical Observability Stack is deployed
- `gnbsim` where the gNB simulator is deployed
- `user-plane` where all the user plane function is deployed

You have learned how to:

- view the logs for the various functions
- manage the integrations between deployed functions
- run a simulation testing data flow through the 5g core
- view the metrics produced by the 5g core

## 11. Cleaning up

Juju makes it simple to cleanly remove all the deployed applications by simply removing the model itself. To completely remove all deployments, use the following:

```bash
juju destroy-controller --destroy-all-models sdcore --destroy-storage
```

You can now proceed to remove Juju itself on the `juju-controller` VM:

```console
sudo snap remove juju
```

MicroK8s can also be removed from each cluster as follows:

```console
sudo snap remove microk8s
```

You may wish to reboot the Multipass VMs to ensure no residual network configurations remain.

Multipass VMs also can be deleted from the host machine:

```console
multipass delete --all
```

If required, all the VMs can be permanently removed:

```console
multipass purge
```

Remove the initially created projects and store pool from the LXD:

```console
lxc project delete sdcore
lxc project delete multipass
lxc storage delete sdcore
```

Delete the local network bridges that are created for LXD. Remove the configuration file from the host machine and apply the network configuration:

```console
sudo rm /etc/netplan/99-sdcore-networks.yaml
sudo netplan apply
```