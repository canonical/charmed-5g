# Mastering

In this tutorial, we will deploy and run the SD-Core 5G core network in a single server following Control and User Plane Separation (CUPS) principles. The radio and cell phone simulator will also be deployed on an isolated cluster. [Multipass](https://multipass.run/) is used to create separate VMs connected with [LXD](https://ubuntu.com/lxd) networking.

## 1. Prepare Host

This tutorial requires:
- Ubuntu OS is running on host (preferably 22.04 or higher)
- It has at least one NIC connected to internet 
### Networks

The following networks will be used to create test  environment.

| Name | Subnet | Gateway IP |
| ---- | ------ | ---------- |
| `management` | 10.201.0.0/24 | 10.201.0.1 |
| `access`     | 10.202.0.0/24 | 10.202.0.1 |
| `core`       | 10.203.0.0/24 | 10.203.0.1 |
| `ran`        | 10.204.0.0/24 | 10.204.0.1 |


Create local network bridges to be used by LXD by adding the below config under `/etc/netplan/config.yaml`. `enp4s0` is the sample interface used in this tutorial and replace it with a proper interface name according to your environment.

```yaml
# /etc/netplan/config.yaml
network:
  ethernets:
    enp4s0:
      dhcp4: no
      dhcp6: no
  bridges:
    mgmt-br:
      addresses:
        - 10.201.0.14/24
      routes:
        - to: default
          via: 10.201.0.1
          metric: 110
      interfaces:
        - vlan201
    access-br:
      addresses:
        - 10.202.0.14/24
      routes:
        - to: 10.204.0.0/24
          via: 10.202.0.1
      interfaces:
        - vlan202
    core-br:
      addresses:
        - 10.203.0.14/24
      routes:
        - to: default
          via: 10.203.0.1
          metric: 203
      interfaces:
        - vlan203
    ran-br:
      addresses:
        - 10.204.0.14/24
      routes:
        - to: 10.202.0.0/24
          via: 10.204.0.1
      interfaces:
        - vlan204

  vlans:
    vlan201:
      id: 201
      link: enp4s0
    vlan202:
      id: 202
      link: enp4s0
    vlan203:
      id: 203
      link: enp4s0
    vlan204:
      id: 204
      link: enp4s0

  version: 2
```

```bash
sudo netplan apply
```

### Install and Configure LXD

Install LXD:
```console
$ sudo snap install lxd
```
Pay attention while setting the size of new loop device and local network bridge. 
- Loop device size should be enough to launch all VMs
- New lxd bridge should not be created.

Use default settings for other configurations.
```console
$ lxd init
Would you like to use LXD clustering? (yes/no) [default=no]: 
Do you want to configure a new storage pool? (yes/no) [default=yes]: 
Name of the new storage pool [default=default]: 
Name of the storage backend to use (dir, lvm, zfs, btrfs, ceph) [default=zfs]: 
Create a new ZFS pool? (yes/no) [default=yes]: 
Would you like to use an existing empty block device (e.g. a disk or partition)? (yes/no) [default=no]: 
Size in GiB of the new loop device (1GiB minimum) [default=30GiB]: 165GiB
Would you like to connect to a MAAS server? (yes/no) [default=no]: 
Would you like to create a new local network bridge? (yes/no) [default=yes]: no
What IPv4 address should be used? (CIDR subnet notation, “auto” or “none”) [default=auto]: 
What IPv6 address should be used? (CIDR subnet notation, “auto” or “none”) [default=auto]: 
Would you like the LXD server to be available over the network? (yes/no) [default=no]: 
Would you like stale cached images to be updated automatically? (yes/no) [default=yes]: 
Would you like a YAML "lxd init" preseed to be printed? (yes/no) [default=no]: 
```

### Install and configure Multipass

Install Multipass and set LXD as local driver:

```console
$ sudo snap install multipass
$ sudo multipass set local.driver=lxd
```
Wait a few seconds if you get the output: `set failed: cannot connect to the multipass socket` and retry to set local driver again.

Check if local driver is set to lxd properly.
```console
$ sudo multipass get local.driver
lxd
```

Connect multipass to LXD.
```console
$ sudo snap connect multipass:lxd lxd
```

Check multipass connections:
```console
$ sudo snap connections multipass
```

Set Multipass passphrase and authenticate:
```console
$ sudo multipass set local.passphrase
Please enter passphrase: <your_passhrase>
Please re-enter passphrase: <your_passhraseo> 

$ multipass authenticate
Please enter passphrase: <your_passhrase>
```

## 2. Create VMs 

To complete this tutorial, you will need seven virtual machines with access to the networks as follows.

| Machine                              | CPUs | RAM | Disk | Networks                       |
|--------------------------------------|------|-----|------|--------------------------------|
| DNS Server                           | 1    | 1g  | 15g  | `management`                   |
| Control Plane Kubernetes Cluster     | 4    | 8g  | 40g  | `management`                   |
| User Plane Kubernetes Cluster        | 4    | 4g  | 20g  | `management`, `access`, `core` |
| Juju Controller + Kubernetes Cluster | 2    | 6g  | 40g  | `management`                   |
| gNB Simulator Kubernetes Cluster     | 2    | 4g  | 20g  | `management`, `ran`            |
| RAN Access Router                    | 1    | 1g  | 15g  | `management`, `ran` , `access` |
| Core Router                          | 1    | 1g  | 15g  | `management`, `core`           |


Create VMs with Multipass:

```console
$ multipass launch -c 1 -m 1G -d 15G -n dns --network mgmt-br jammy
$ multipass launch -c 4 -m 8G -d 40G -n control-plane --network mgmt-br jammy
$ multipass launch -c 4 -m 4G -d 20G -n user-plane  --network mgmt-br --network core-br --network access-br jammy
$ multipass launch -c 4 -m 6G -d 40G -n juju-controller --network mgmt-br jammy
$ multipass launch -c 2 -m 4G -d 20G -n gnbsim --network mgmt-br --network ran-br jammy
$ multipass launch -c 1 -m 1G -d 15G -n ran-access-router  --network mgmt-br --network ran-br --network access-br jammy
$ multipass launch -c 1 -m 1G -d 15G -n core-router --network mgmt-br --network core-br jammy
```

Wait till all VMs become `Running` status.

```console
Name                    State             IPv4             Image
juju-controller         Running           10.231.204.5     Ubuntu 22.04 LTS
                                          10.201.0.104
                                          10.1.49.0
core-router             Running           10.231.204.200   Ubuntu 22.04 LTS
                                          10.201.0.114
                                          10.203.0.1
control-plane           Running           10.231.204.202   Ubuntu 22.04 LTS
                                          10.201.0.101
                                          10.1.9.0
dns                     Running           10.231.204.96    Ubuntu 22.04 LTS
                                          10.201.0.100
gnbsim                  Running           10.231.204.24    Ubuntu 22.04 LTS
                                          10.201.0.103
                                          10.204.0.100
                                          10.1.143.192
ran-access-router       Running           10.231.204.220   Ubuntu 22.04 LTS
                                          10.201.0.110
                                          10.204.0.1
                                          10.202.0.1
user-plane              Running           10.231.204.121   Ubuntu 22.04 LTS
                                          10.201.0.102
                                          10.203.0.100
                                          10.202.0.100
                                          10.1.112.128
```

### IP Addresses

The following IP addresses are used in this tutorial and must be present in the DNS Server that all hosts are using.

| Name | IP Address   | Purpose |
| ---- |--------------| ------- |
| `juju-controller.mgmt` | 10.201.0.104 | Management address for Juju machine |
| `control-plane.mgmt` | 10.201.0.101 | Management address for control plane cluster machine |
| `user-plane.mgmt` | 10.201.0.102 | Management address for user plane cluster machine |
| `gnbsim.mgmt` | 10.201.0.103 | Management address for the gNB Simulator cluster machine |
| `api.juju-controller.mgmt` | 10.201.0.50  | Juju controller address |
| `cos.mgmt` | 10.201.0.51  | Canonical Observability Stack address |
| `amf.mgmt` | 10.201.0.52  | Externally reachable control plane endpoint for the AMF |
| `control-plane-nms.control-plane.mgmt` | 10.201.0.53  | Externally reachable control plane endpoint for the NMS |
| `upf.mgmt` | 10.201.0.200 | Externally reachable control plane endpoint for the UPF |


### Install DNS Server

First, replace the content of `/etc/netplan/50-cloud-init.yaml` as following to configure mgmt interface ip address as `10.201.0.100`.

```yaml
network:
    ethernets:
        enp5s0:
            dhcp4: true
        enp6s0:
            dhcp4: false
            addresses:
              - 10.201.0.100/24
    version: 2
```

```console
$ sudo netplan apply
```

Install DNS server:
```console
$ multipass exec dns -- bash
$ sudo apt update
$ sudo apt install dnsmasq -y
$ sudo systemctl disable systemd-resolved
$ sudo systemctl stop systemd-resolved
$ sudo systemctl restart dnsmasq
$ sudo systemctl status  dnsmasq
```

Configure dnsmasq:
```console
$ cat << EOF | sudo tee -a /etc/dnsmasq.conf
no-resolv
server=8.8.8.8
server=8.8.4.4
domain=mgmt
EOF
```

Add records under /etc/hosts:
```console
$ cat << EOF | sudo tee -a /etc/hosts
10.201.0.104   juju-controller.mgmt
10.201.0.101   control-plane.mgmt
10.201.0.102   user-plane.mgmt
10.201.0.103   gnbsim.mgmt
10.201.0.50    api.juju-controller.mgmt
10.201.0.51    cos.mgmt
10.201.0.52    amf.mgmt
10.201.0.53    control-plane-nms.control-plane.mgmt
10.201.0.200   upf.mgmt
EOF
```

Restart DNS server:
```console
$ sudo systemctl reload dnsmasq
```

Test the DNS resolution.
```console
$ dig upf.mgmt
; <<>> DiG 9.18.18-0ubuntu0.22.04.1-Ubuntu <<>> upf.mgmt
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 46312
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;upf.mgmt.			IN	A

;; ANSWER SECTION:
upf.mgmt.		0	IN	A	10.201.0.200

;; Query time: 0 msec
;; SERVER: 127.0.0.53#53(127.0.0.53) (UDP)
;; WHEN: Fri Dec 15 15:07:40 +03 2023
;; MSG SIZE  rcvd: 53
``` 

### Configure network of VMs to add DNS server and routes

#### User-plane VM

Log in to DNS VM.
```console
$ multipass exec dns -- bash
```

Configure ip address for mgmt, core and access interfaces, add nameservers for "mgmt" interface and add route between Access to RAN network by replacing the content of /etc/netplan/50-cloud-init.yaml as following:

 ```yaml
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
 ```

Then, run the command to make the new configuration become active.
 ```console
 sudo netplan apply
 ```

Check the current DNS server. Repeat this step in all VMs after setting DNS server.
```console
$ resolvectl

Link 3 (enp6s0)
    Current Scopes: DNS
         Protocols: +DefaultRoute +LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
Current DNS Server: 10.201.0.100
       DNS Servers: 10.201.0.100
        DNS Domain: mgmt
```

Check the route from Access interface to RAN network.
```console
$ ip route
...
10.204.0.0/24 via 10.202.0.1 dev enp8s0 proto static
```

#### Control-plane VM

Log in to control-plane VM.

```console
$ multipass exec control-plane -- bash
```

Configure ip address and nameservers for "mgmt" interface by replacing the content of /etc/netplan/50-cloud-init.yaml as following:

```yaml
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
```

Then, run the command to make the new configuration become active.
 ```console
$ sudo netplan apply
 ```

#### Gnbsim VM

Log in to gnbsim VM.

```console
$ multipass exec gnbsim -- bash
```

Configure ip address for mgmt and ran interfaces, add nameservers for "mgmt" interface and add route between RAN to access network by replacing the content of /etc/netplan/50-cloud-init.yaml as following:

```yaml
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
```

Run the command to make the new configuration become active.
 ```console
 sudo netplan apply
 ```

#### Juju-controller VM

Log in to juju-controller VM.

```console
$ multipass exec juju-controller -- bash
```

Configure ip address and nameservers for "mgmt" interface by replacing the content of /etc/netplan/50-cloud-init.yaml as following:

```yaml
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
```

Run the command to make the new configuration become active.
 ```console
 sudo netplan apply
 ```


#### RAN-access-router VM

Log in to ran-access-router VM.

```console
$ multipass exec ran-access-router -- bash
```

Configure ip address for mgmt, ran and access interfaces by replacing the content of /etc/netplan/50-cloud-init.yaml as following:

```yaml
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
```

The `access-gateway-ip` is expected to forward the packets from the `access-interface` to the `gnb-subnet`.

Set up IP forwarding

```console
echo net.ipv4.ip_forward=1 | sudo tee /etc/sysctl.conf
sudo sysctl -w net.ipv4.ip_forward=1
```

#### Core-router VM

Log in to core-router VM.

```console
$ multipass exec core-router -- bash
```

Configure ip address for mgmt and core interfaces by replacing the content of /etc/netplan/50-cloud-init.yaml as following:

```yaml
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
```

Set up ip forwarding and NAT:

```console
cat << EOF | sudo tee /etc/rc.local
#!/bin/bash
iptables -t nat -A POSTROUTING -o enp5s0 -j MASQUERADE -s 10.203.0.0/16
EOF
sudo chmod +x /etc/rc.local
sudo /etc/rc.local
echo net.ipv4.ip_forward=1 | sudo tee /etc/sysctl.conf
sudo sysctl -w net.ipv4.ip_forward=1
```

## 3. Configure VMs for SD-Core Deployment

This section covers how to install Juju and MicroK8s to act as the infrastructure for SD-Core. Besides it also set up ssh keys between VMs.

### Prepare SD-Core Control Plane VM

All commands in this section are run on the `control-plane` VM.

Login `control-plane` VM.

```console
multipass exec control-plane -- bash
```

Install Microk8s.

```console
sudo snap install microk8s --channel=1.27-strict/stable
sudo microk8s enable hostpath-storage
sudo usermod -a -G snap_microk8s $USER
newgrp snap_microk8s
```

The control plane needs to expose two services: the AMF and the NMS. In this step, we enable the MetalLB add on in Microk8s, and give it a range of two IP addresses:

```console
sudo microk8s enable metallb:10.201.0.52-10.201.0.53
```

Generate ssh key pair and copy public key to `juju-controller` VM. 

Create key pair.
```console
$ ssh-keygen 
Generating public/private rsa key pair.
Enter file in which to save the key (/home/ubuntu/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/ubuntu/.ssh/id_rsa
Your public key has been saved in /home/ubuntu/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:7KO+OPL3hmkmMo8RyidTC8s31Z9G3ExqB7jTX1fg6/c ubuntu@cplane
The key's randomart image is:
+---[RSA 3072]----+
|              .  |
|        .    . . |
|       . . .  . .|
|      ..+ *    ..|
| ... . +S* + ... |
|o.+.o  .= + ...  |
|.*.=   oo+ .  . .|
|  Oo+.*.o.     ..|
|  .*+B++.       E|
+----[SHA256]-----+
```

Open and copy the public key.
```console
$ cat .ssh/id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDmbwQqKTooFMOgZNTtnCFrQP+DYxiMzX9+Xhuw9i8aQ+bcFSNicmTCZf5JVxRrLprrE/Ta9OOhWF4LfPX3qAT+zDJe7uKw6vu+G8HWu2fqWCiK+7ur+d5v9LZPeHGJyuQ3SoLF61ao+zeHlaEoaYvVQTiXuvgVEKqdEaSii08aodGuv2qkdUuij7SDQsrLoIkTs3zFDjQZDdd2gYXRNLCqH33nEZ7L47kMsMVTYj5YK9804iuP/6x83f9WlLsEgJ+I7fpMLX7mno3A1Ef6nAA0/FvN2JMc4CU+L+iMd9UENR8zUDhqifd08YyOvSlmbqg8WhqoCYrDcZ3SNBiYE8/th6O+ppERCjqxBxnBnWh+qGokIGA74qRJtXpdVbneg4eR3Ehy2j4EdUhG0uEgz4/H4gPF+qDvLj8HbODt3TSCdCh7qETQVVJT5vmOoOq1AgvNgX23iRZPTesHiNKq8Z4upeCZfG5Z2Zy2MtULBbNK5Q7Gy5eY5tN3XRohM61hh5k= ubuntu@control-plane
```

Login `juju-controller` VM and add the public key to under .ssh/authorized_keys file.

```console
multipass exec juju-oontroller -- bash
```

Add `conrol-plane` VM public key under `juju-controller` VMs authorized_keys.
```console
cat << EOF | sudo tee -a .ssh/authorized_keys
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDmbwQqKTooFMOgZNTtnCFrQP+DYxiMzX9+Xhuw9i8aQ+bcFSNicmTCZf5JVxRrLprrE/Ta9OOhWF4LfPX3qAT+zDJe7uKw6vu+G8HWu2fqWCiK+7ur+d5v9LZPeHGJyuQ3SoLF61ao+zeHlaEoaYvVQTiXuvgVEKqdEaSii08aodGuv2qkdUuij7SDQsrLoIkTs3zFDjQZDdd2gYXRNLCqH33nEZ7L47kMsMVTYj5YK9804iuP/6x83f9WlLsEgJ+I7fpMLX7mno3A1Ef6nAA0/FvN2JMc4CU+L+iMd9UENR8zUDhqifd08YyOvSlmbqg8WhqoCYrDcZ3SNBiYE8/th6O+ppERCjqxBxnBnWh+qGokIGA74qRJtXpdVbneg4eR3Ehy2j4EdUhG0uEgz4/H4gPF+qDvLj8HbODt3TSCdCh7qETQVVJT5vmOoOq1AgvNgX23iRZPTesHiNKq8Z4upeCZfG5Z2Zy2MtULBbNK5Q7Gy5eY5tN3XRohM61hh5k= ubuntu@control-plane
EOF
```

Export the Kubernetes configuration and copy that to the controller:

```console
sudo microk8s.config > control-plane-cluster.yaml
scp control-plane-cluster.yaml juju-controller.mgmt:
```

Set alias for kubectl:
```console
cat << EOF | sudo tee -a ~/.bashrc
alias kubectl="microk8s.kubectl"
EOF
source ~/.bashrc
```

Change coredns configmap by adding local dns server which is `10.201.0.100`.

Append the following lines in configmap/coredns at the end of Corefile section.

```yaml
    mgmt:53 {
         errors
         cache 30
         forward . 10.201.0.100
    }
```

Edit configmap by running:
```console
kubectl -n kube-system edit configmap/coredns
```

After modification, the configmap looks like below yaml:

```yaml
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health {
          lameduck 5s
        }
        ready
        log . {
          class error
        }
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . 8.8.8.8 8.8.4.4
        cache 30
        loop
        reload
        loadbalance
    }
    mgmt:53 {
         errors
         cache 30
         forward . 10.201.0.100
    }
kind: ConfigMap
```

### Prepare SD-Core User Plane VM

All commands in this section are run on the `user-plane` VM.

Install MicroK8s, configure MetalLB to expose 1 IP address for the UPF (`10.201.0.200`), and add the Multus plugin:

```console
sudo snap install microk8s --channel=1.27-strict/stable
sudo microk8s enable hostpath-storage
sudo microk8s enable metallb:10.201.0.200-10.201.0.200
sudo microk8s addons repo add community \
    https://github.com/canonical/microk8s-community-addons \
    --reference feat/strict-fix-multus
sudo microk8s enable multus
sudo usermod -a -G snap_microk8s $USER
newgrp snap_microk8s
```

Generate ssh key pair and copy the public key to `juju-controller` VM.

Create key pair.
```console
$ ssh-keygen 
Generating public/private rsa key pair.
Enter file in which to save the key (/home/ubuntu/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/ubuntu/.ssh/id_rsa
Your public key has been saved in /home/ubuntu/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:7KO+OPL3hmkmMo8RyidTC8s31Z9G3ExqB7jTX1fg6/c ubuntu@cplane
The key's randomart image is:
+---[RSA 3072]----+
|              .  |
|        .    . . |
|       . . .  . .|
|      ..+ *    ..|
| ... . +S* + ... |
|o.+.o  .= + ...  |
|.*.=   oo+ .  . .|
|  Oo+.*.o.     ..|
|  .*+B++.       E|
+----[SHA256]-----+
```

Open and copy the public key.
```console
$ cat .ssh/id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDF4f0wB/ahrHqmUBiM+7x+gg04V1G3rTMK1VhKUvnntdRqYaEG2iR7V4f0EkwPscHtToPSV9qTVnHir5grtxxexTmfBr4NyiLcrix33I28w0ttRh4Y6k3jTzTlZ7ArN69abgiF2oz9nbzGEqiXho4V39IwSbGyzQzcduWNLMpA5ftXFDT4lB59hDJCPDlRSPe6wxBZ6doNPQemkudoCXXCGbdgCrZlzXFX4G0xoFQqyGCkPgDINpYGad+SvwU7j2JsmsaRNE+GF4S3cjNRChr7wlfijgXpi2a4Psiceg0LiyFMdALQYofQNdFbiUBlyp3lxNv6ph55QerOAO7l5mMdrGIL4xdhDqZnrWUgL5gI8sywQZMOfjm5kbkJiGKYggM8scE4ntWakxXpLcKiiXUvj9Ccou/+xVFK9YFgDP9c/75Y4DJgUrwkCmcJUUqqX94rjUsvMWmSruTKmSAatq1HFCL1W6dGMr4bj9bl7fmPhIYeBOJM0O1VClx0Umb/D2s= ubuntu@user-plane
```

Login `juju-controller` VM and add the public key to under .ssh/authorized_keys file.

```console
multipass exec juju-oontroller -- bash
```

Add `conrol-plane` VM public key under `juju-controller` VMs authorized_keys.
```console
cat << EOF | sudo tee -a .ssh/authorized_keys
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDF4f0wB/ahrHqmUBiM+7x+gg04V1G3rTMK1VhKUvnntdRqYaEG2iR7V4f0EkwPscHtToPSV9qTVnHir5grtxxexTmfBr4NyiLcrix33I28w0ttRh4Y6k3jTzTlZ7ArN69abgiF2oz9nbzGEqiXho4V39IwSbGyzQzcduWNLMpA5ftXFDT4lB59hDJCPDlRSPe6wxBZ6doNPQemkudoCXXCGbdgCrZlzXFX4G0xoFQqyGCkPgDINpYGad+SvwU7j2JsmsaRNE+GF4S3cjNRChr7wlfijgXpi2a4Psiceg0LiyFMdALQYofQNdFbiUBlyp3lxNv6ph55QerOAO7l5mMdrGIL4xdhDqZnrWUgL5gI8sywQZMOfjm5kbkJiGKYggM8scE4ntWakxXpLcKiiXUvj9Ccou/+xVFK9YFgDP9c/75Y4DJgUrwkCmcJUUqqX94rjUsvMWmSruTKmSAatq1HFCL1W6dGMr4bj9bl7fmPhIYeBOJM0O1VClx0Umb/D2s= ubuntu@user-plane
EOF
```

Export the Kubernetes configuration and copy that to the controller:

```console
sudo microk8s.config > user-plane-cluster.yaml
scp user-plane-cluster.yaml juju-controller.mgmt:
```

Set alias for kubectl:
```console
cat << EOF | sudo tee -a ~/.bashrc
alias kubectl="microk8s.kubectl"
EOF
source ~/.bashrc
```

Change coredns configmap by adding local dns server which is `10.201.0.100`.

Append the following lines in configmap/coredns at the end of Corefile section.

```yaml
    mgmt:53 {
         errors
         cache 30
         forward . 10.201.0.100
    }
```

Edit configmap by running:
```console
kubectl -n kube-system edit configmap/coredns
```

After modification, the configmap looks like the one below:

```yaml
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health {
          lameduck 5s
        }
        ready
        log . {
          class error
        }
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . 8.8.8.8 8.8.4.4
        cache 30
        loop
        reload
        loadbalance
    }
    mgmt:53 {
         errors
         cache 30
         forward . 10.201.0.100
    }
kind: ConfigMap
```

In this guide, the following network interfaces are available on the SD-Core User Plane machine:

| Interface Name    | Purpose                                                                                                                                                           |
|-------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| enp6s0            | internal Kubernetes management interface. This maps to the `management` subnet.                                                                                   |
| enp7s0            | core interface. This maps to the `core` subnet.                                                                                                                   |
| enp8s0            | access interface. This maps to the `access` subnet. Note that internet egress is required here and routing tables are already set to route gNB generated traffic. |

Now we create the MACVLAN bridges for enp7s0 and enp7s0. These instructions are put into a file that is executed on reboot so the interfaces will come back.

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

### Prepare gNB Simulator VM

All commands in this section are run on the `gnbsim` VM.

Install MicroK8s and add the Multus plugin:

```console
sudo snap install microk8s --channel=1.27-strict/stable
sudo microk8s enable hostpath-storage
sudo microk8s addons repo add community \
    https://github.com/canonical/microk8s-community-addons \
    --reference feat/strict-fix-multus
sudo microk8s enable multus
sudo usermod -a -G snap_microk8s $USER
newgrp snap_microk8s
```

Generate ssh key pair and copy the public key to `juju-controller` VM.

Create key pair.
```console
$ ssh-keygen 
Generating public/private rsa key pair.
Enter file in which to save the key (/home/ubuntu/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/ubuntu/.ssh/id_rsa
Your public key has been saved in /home/ubuntu/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:7KO+OPL3hmkmMo8RyidTC8s31Z9G3ExqB7jTX1fg6/c ubuntu@cplane
The key's randomart image is:
+---[RSA 3072]----+
|              .  |
|        .    . . |
|       . . .  . .|
|      ..+ *    ..|
| ... . +S* + ... |
|o.+.o  .= + ...  |
|.*.=   oo+ .  . .|
|  Oo+.*.o.     ..|
|  .*+B++.       E|
+----[SHA256]-----+
```

Open and copy the public key.
```console
$ cat .ssh/id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCs6xKaPLaOjX577VE47UuLlsu1bK9Otar2oJZOLFrQuncDEm5GBQrqGxzN1OLfY9zZH4js7gnAFRhGq6R7D8CQckDVZx91g5aawnU4lgaT3ZfgyjNVUjOh7AMjn9aK4Foolp8aZrDgaSR/FFvMAOoBYkvAFBTpFRd0RAeKYxowiUWVo46AwMP/SyhhosV40ahwOm9MXUIvBqlV877ZWUzUNWOHNT3hpB1r286TyViuSZOGjCm8BXooS2zlTnCE2EsQtmzbb5AuqtuI6UBLUtvv/47605caVyGV+aPDWHK7pz014aL7Sh1DduDcs046igGTjNjpGhkCrfASAkaa66I7KEZJfpLa/kwpnqv8bn6mvuXNvaD/+NcFdb1xyxk8bfBl9qsfJ1gKrSide00GwXOLy0aU7YH9KoQavfhF4/OX9q65kl1SfQeukbRRHWhltg8f6MXr+7WBlk7FcAzgCiQZMUhxpZsAv3FQQgiZCN4G20K/2pajEzuEr7NEWLG7W2U= ubuntu@gnbsim
```

Login `juju-controller` VM and add the public key to under .ssh/authorized_keys file.

```console
multipass exec juju-controller -- bash
```

Add `conrol-plane` VM public key under `juju-controller` VMs authorized_keys.
```console
cat << EOF | sudo tee -a .ssh/authorized_keys
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCs6xKaPLaOjX577VE47UuLlsu1bK9Otar2oJZOLFrQuncDEm5GBQrqGxzN1OLfY9zZH4js7gnAFRhGq6R7D8CQckDVZx91g5aawnU4lgaT3ZfgyjNVUjOh7AMjn9aK4Foolp8aZrDgaSR/FFvMAOoBYkvAFBTpFRd0RAeKYxowiUWVo46AwMP/SyhhosV40ahwOm9MXUIvBqlV877ZWUzUNWOHNT3hpB1r286TyViuSZOGjCm8BXooS2zlTnCE2EsQtmzbb5AuqtuI6UBLUtvv/47605caVyGV+aPDWHK7pz014aL7Sh1DduDcs046igGTjNjpGhkCrfASAkaa66I7KEZJfpLa/kwpnqv8bn6mvuXNvaD/+NcFdb1xyxk8bfBl9qsfJ1gKrSide00GwXOLy0aU7YH9KoQavfhF4/OX9q65kl1SfQeukbRRHWhltg8f6MXr+7WBlk7FcAzgCiQZMUhxpZsAv3FQQgiZCN4G20K/2pajEzuEr7NEWLG7W2U= ubuntu@gnbsim
EOF
```

Export the Kubernetes configuration and copy that to the controller:

```console
sudo microk8s.config > gnb-cluster.yaml
scp gnb-cluster.yaml juju-controller.mgmt:
```

Set alias for kubectl:
```console
cat << EOF | sudo tee -a ~/.bashrc
alias kubectl="microk8s.kubectl"
EOF
source ~/.bashrc
```

Change coredns configmap by adding local dns server which is `10.201.0.100`.

Append the following lines in configmap/coredns at the end of Corefile section.

```yaml
    mgmt:53 {
         errors
         cache 30
         forward . 10.201.0.100
    }
```

Edit configmap by running:
```console
kubectl -n kube-system edit configmap/coredns
```

After modification, the configmap looks like the one below:

```yaml
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health {
          lameduck 5s
        }
        ready
        log . {
          class error
        }
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . 8.8.8.8 8.8.4.4
        cache 30
        loop
        reload
        loadbalance
    }
    mgmt:53 {
         errors
         cache 30
         forward . 10.201.0.100
    }
kind: ConfigMap
```

In this guide, the following network interfaces are available on the gNB Simulator machine:

| Interface Name | Purpose                                                                         |
|----------------|---------------------------------------------------------------------------------|
| enp6s0           | internal Kubernetes management interface. This maps to the `management` subnet. |
| enp7s0           | ran interface. This maps to the `ran` subnet.                                   |

Now we create the MACVLAN bridges for enp7s0, and label them accordingly:

```console
cat << EOF | sudo tee /etc/rc.local
#!/bin/bash

sudo ip link add ran link enp7s0 type macvlan mode bridge
sudo ip link set dev ran up
EOF
sudo chmod +x /etc/rc.local
sudo /etc/rc.local
```

### Prepare Juju Controller VM

Begin by installing MicroK8s to hold the Juju controller, and configure MetalLB to expose one IP address for the controller (`10.201.0.50`), and one for the Canonical Observability Stack (`10.201.0.51)`.

```console
sudo snap install microk8s --channel=1.27-strict/stable
sudo microk8s enable hostpath-storage
sudo microk8s enable metallb:10.201.0.50-10.201.0.51
sudo usermod -a -G snap_microk8s $USER
newgrp snap_microk8s
```

Install Juju and bootstrap the controller to the local Microk8s install as a load balancer service. This will expose the Juju controller on the first allocated MetalLB address.

```console
mkdir -p ~/.local/share/juju
sudo snap install juju --channel=3.1/stable
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

Set alias for kubectl:
```console
cat << EOF | sudo tee -a ~/.bashrc
alias kubectl="microk8s.kubectl"
EOF
source ~/.bashrc
```

Change coredns configmap by adding local dns server which is `10.201.0.100`.

Append the following lines in configmap/coredns at the end of `Corefile` section.

```yaml
    mgmt:53 {
         errors
         cache 30
         forward . 10.201.0.100
    }
```

Edit configmap by running:
```console
kubectl -n kube-system edit configmap/coredns
```

After modification, the configmap looks like the one below:

```yaml
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health {
          lameduck 5s
        }
        ready
        log . {
          class error
        }
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . 8.8.8.8 8.8.4.4
        cache 30
        loop
        reload
        loadbalance
    }
    mgmt:53 {
         errors
         cache 30
         forward . 10.201.0.100
    }
kind: ConfigMap
```
You may now proceed to deploy the SD-Core Control Plane and SD-Core User Plane

## 4. Deploy SD-Core Control Plane

The following steps build on the Juju controller which was bootstrapped and knows how to manage the SD-Core Control Plane Kubernetes cluster.

First, create a Juju overlay file that specifies the Access and Mobility Management Function (AMF) host name and IP address for sharing with the radios. This host name must be resolvable by the gNB and the IP address must be reachable and resolve to the AMF unit. In the bootstrap step, we set the Control Plane MetalLB range to start at `10.201.0.52`, so that is what we use in the configuration.

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
juju deploy sdcore-control-plane-k8s --trust --channel=edge --overlay control-plane-overlay.yaml
```

Expose the Software as a Service offer for the AMF.

```console
juju offer control-plane.amf:fiveg-n2
```

Configure the NMS to be served as `control-plane.mgmt`:

```console
juju config traefik-k8s external_hostname=control-plane.mgmt
```

### Checkpoint

You should be able to see the AMF External load balancer service in Kubernetes. Log into the `control-plane` VM and execute the following command:

```console
multipass exec control-plane -- bash
kubectl get services -A | grep LoadBalancer
```

This will show output similar to the following, indicating:
- The AMF is exposed on 10.201.0.100 SCTP port 38412
- The NMS is exposed on 10.201.0.101 TCP ports 80 and 443

These IP addresses came from MetalLB and were configured in the bootstrap step.

```
control-plane    amf-external  LoadBalancer  10.152.183.179  10.201.0.52   38412:30408/SCTP
control-plane    traefik-k8s   LoadBalancer  10.152.183.28   10.201.0.53   80:32349/TCP,443:31925/TCP
```

## 5. Deploy SD-Core User Plane

Create a Juju model to represent the User Plane, using the cloud `user-plane-cluster`, which was created in the Bootstrap step. The following parameters can be set:

- `access-gateway-ip`: this is the IP address of the gateway that knows how to route traffic from the UPF towards the gNB subnet
- `access-interface`: the name of the MACVLAN interface on the Kubernetes host cluster to bridge to the `access` subnet
- `access-ip`: the IP address for the UPF to use on the `access` subnet
- `core-gateway-ip`: this is the IP address of the gateway that knows how to route traffic from the UPF towards the internet
- `core-interface`: the name of the MACVLAN interface on the Kubernetes host cluster to bridge to the `core` subnet
- `core-ip`: the IP address for the UPF to use on the `core` subnet
- `gnb-subnet`: the subnet CIDR where the gNB radios are reachable.


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
      gnb-subnet: 10.204.0.0/24
      external-upf-hostname: upf.mgmt
EOF
```

Next, we create a Juju model to represent the User Plane, using the cloud `user-plane-cluster`, which was created in the Bootstrap step.

```console
juju add-model user-plane user-plane-cluster
```

Deploy user plane bundle:

```console
juju deploy sdcore-user-plane-k8s --trust --channel=edge --overlay upf-overlay.yaml
```

Now expose the UPF service offering with Juju.

```console
juju offer user-plane.upf:fiveg_n4
```

### Checkpoint

You should be able to see the UPF External load balancer service in Kubernetes. Log into the user plane cluster and execute the following command:

```console
microk8s.kubectl get services -A | grep LoadBalancer
```

This should produce output similar to the following indicating that the PFCP agent of the UPF is exposed on 10.201.0.200 UDP port 8805
```
user-plane  upf-external  LoadBalancer  10.152.183.126  10.201.0.200  8805:31101/UDP
```

## 6. Deploy the gNB Simulator

Create a new Juju model named `gnbsim` on the Juju controller cloud:

```console
juju add-model gnbsim gnb-cluster
```

Deploy the simulator to the gnbsim cluster. The simulator needs to know the following:

- `gnb-interface`: the name of the MACVLAN interface to use on the host
- `gnb-ip-address`: the IP address to use on the gnb interface
- `icmp-packet-destination`: the target IP address to ping. If there is no egress to the internet on your core network, any IP that is reachable from the UPF should work.
- `upf-gateway`: the IP address of the gateway between the RAN and Access networks
- `upf-subnet`: subnet where the UPFs are located (also called Access network)

```console
juju deploy sdcore-gnbsim-k8s gnbsim --channel=edge \
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

Still on the Juju controller, switch to the control-plane model.

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

The next steps are performed using a web browser to navigate to the NMS at http://control-plane-nms.control-plane.mgmt/

```{note}
The computer that is running the web browser must also use the same DNS server as the rest of the environment !!
```

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

We will integrate the 5G core network with the Canonical Observability Stack (COS). All commands are to be executed on the Juju controller node.

### Deploy the `cos-lite` bundle

Create a new Juju model named `cos` on the Juju controller cloud:

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

First, offer the following integrations from Prometheus and Loki for use in other models:

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

Retrieve the Grafana admin password:

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

## Checkpoint

In your browser, navigate to the URL from the output (`https://10.201.0.51/cos-grafana`). Login using the "admin" username and the admin password provided in the last command. Click on "Dashboards" -> "Browse" and select "5G Network Overview".

This dashboard presents an overview of your 5G Network status. Keep this page open, we will revisit it shortly.

```{image} ../images/grafana_5g_dashboard_sim_before.png
:alt: Initial Grafana dashboard showing UPF status
:align: center
```

## 9. Run the 5G simulation

On the juju controller, switch to the gnbsim model.
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

## Checkpoint

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
2023-11-30T16:43:40Z [INFO][GNBSIM][GNodeB][ControlPlaneTransport] Connected to AMF, AMF IP: 10.201.0.100 AMF Port: 38412
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
kubectl logs -n control-plane -c amf amf-0 --tail 70
kubectl logs -n control-plane -c ausf ausf-0 --tail 70
kubectl logs -n control-plane -c nrf nrf-0 --tail 70
kubectl logs -n control-plane -c nssf nssf-0 --tail 70
kubectl logs -n control-plane -c pcf pcf-0 --tail 70
kubectl logs -n control-plane -c smf smf-0 --tail 70
kubectl logs -n control-plane -c udm udm-0 --tail 70
kubectl logs -n control-plane -c udr udr-0 --tail 70
```

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
juju destroy-controller --destroy-all-models sdcore
```

You can now proceed to remove Juju itself on the `juju-controller`:

```console
sudo snap remove juju
```

MicroK8s can also be removed from each cluster as follows:

```console
sudo snap remove microk8s
```

You may wish to reboot the multipass VMs to ensure no residual network configurations remain.

Multipass VMs also can be deleted from host machine:

```console
multipass delete --all
```

If required, the VMs can be permanently removed:
```console
multipass purge
```
