# Deploy SD-Core with CUPS

This guide covers how to install a multi-node SD-Core with Control Plane and User Plane Separation (CUPS).

## Requirements

You will need two Kubernetes clusters as follows.  In this how-to, we cover installation and configuration of MicroK8s, however, any compatible Kubernetes cluster can be used.
- `SD-Core Control Plane`, where the full set of control plane functions will be placed (AMF, SMF, etc)
- `SD-Core User Plane`, where the User Plane Function (UPF) will be placed

A separate machine is required for the Juju controller. [`TODO:` note on MetalLB for Juju controller]

In the `SD-Core User Plane` cluster, the following subnets must be present:
- `access`, where the UPF listens for incoming sessions from the gNB
- `core`, where the UPF send out traffic from the gNB towards the internet
- `management`, for Juju and control plane traffic.  Must also have access to the internet for downloading container images, charms, or other updates as needed

[`TODO:` Add note about port security somewhere for the networks]

### Sample Values

For the purpose of this guide, the following values will be used.

#### Networks

| Name | VLAN | Subnet | Gateway IP |
| ---- | ---- | ------ | ---------- |
| management | 1201 | 10.201.0.0/24 | 10.201.0.1 |
| access     | 1202 | 10.202.0.0/24 | 10.202.0.1 |
| core       | 1203 | 10.203.0.0/24 | 10.203.0.1 |

#### IP Addresses

The only Domain Name Server (DNS) entry that is required is for the AMF so that the gNB can be told where to find it [`TODO:` link to gNB interface reference]

| Name | IP Address | Purpose |
| ---- | ---------- | ------- |
| `juju-controller.k8s.priv` | 10.201.0.10 | Management address for Juju machine
| `control-plane.k8s.priv` | 10.201.0.11 | Management address for control plane cluster machine |
| `user-plane.k8s.priv` | 10.201.0.12 | Management address for user plane cluster machine |
| `control-plane.juju.priv` | 10.201.0.101 | Juju controller address |
| `upf.core.priv` | 10.201.0.151 | IP address for UPF to bind to |
| `amf.core.priv` | 10.201.0.201 | Externally reachable control plane endpoint for the AMF |


`````{tab-set}

````{tab-item} Bootstrap

This section covers how to install Juju and MicroK8s to act as the infrastructure for SD-Core, and may be skipped as needed if already done.

----------------------
## SD-Core Control Plane

All commands in this section are run on the SD-Core Control Plane Kubernetes cluster host.

Install MicroK8s and configure MetalLB to expose 1 IP address for the AMF:

```bash
sudo snap install microk8s --channel=1.27-strict/stable
sudo microk8s enable hostpath-storage
sudo microk8s enable metallb:10.201.0.201-10.201.0.201
```

Export the Kubernetes configuration and copy that to the controller:

```bash
sudo microk8s.config > control-plane-cluster.yaml
scp control-plane-cluster.yaml juju-controller.k8s.priv:
```

----------------------
## SD-Core User Plane

All commands in this section are run on the SD-Core User Plane Kubernetes cluster host.

Install MicroK8s, configure MetalLB to expose 1 IP address for the UPF, and add the Multus plugin:

```bash
sudo snap install microk8s --channel=1.27-strict/stable
sudo microk8s enable hostpath-storage
sudo microk8s enable metallb:10.201.0.151-10.201-0.151
sudo microk8s addons repo add community \
    https://github.com/canonical/microk8s-community-addons \
    --reference feat/strict-fix-multus
sudo microk8s enable multus
```

Export the Kubernetes configuration and copy that to the controller:

```bash
sudo microk8s.config > user-plane-cluster.yaml
scp user-plane-cluster.yaml juju-controller.k8s.priv:
```

In this guide, the following network interfaces are available on the SD-Core User Plane machine:

| Interface Name | Purpose |
|----------------|---------|
| ens3           | internal Kubernetes management interface.  This maps to the `management` subnet and must have an IP address |
| ens4           | access interface.  This maps to the `access` subnet, and does not have an IP on the host |
| ens5           | core interface.  This maps to the `core` subnet, and does not have an IP on the host.  Note that internet egress is required here and routing tables must be set to route gNB generated traffic |

----------------------
## Juju Controller

Machine Minimum Requirements
- Ubuntu 22.04
- a minimum of `TODO:` 2  cores
- a minimum of `TODO:` 4  GiB memory
- a minimum of `TODO:` 20  GiB disk space

Begin by installing MicroK8s to hold the Juju controller, and configure MetalLB to expose the one IP address for it.

```bash
sudo snap install microk8s --channel=1.27-strict/stable
sudo microk8s enable hostpath-storage
sudo microk8s enable metallb:10.201.0.101-10.201.0.101
sudo usermod -a -G snap_microk8s $USER
newgrp snap_microk8s
```

Install Juju and bootstrap the controller:

```bash
mkdir -p ~/.local/share/juju
sudo snap install juju --channel=3.1/stable
juju bootstrap microk8s --config controller-service-type=loadbalancer sdcore
```

At this point, the Juju controller is ready to start managing external clouds.  Add the Kubernetes clusters representing the user plane and control plane to Juju.  This is done by using the Kubernetes configuration file generated when setting up the clusters above.

```bash
export KUBECONFIG=control-plane-cluster.yaml
juju add-k8s control-plane-cluster --controller sdcore
export KUBECONFIG=user-plane-cluster.yaml
juju add-k8s user-plane-cluster --controller sdcore
```

You may now proceed to deploy the SD-Core Control Plane and SD-Core User Plane

````


````{tab-item} SD-Core Control Plane

First, create a Juju overlay file that specifies the Access and Mobility Management Function (AMF) host name and IP address for sharing with the radios.  This host name must be resolvable by the gNB and the IP address must be reachable and resolve to the AMF unit.

```bash
cat << EOF > control-plane-overlay.yaml
applications:
  amf:
    options:
      external-amf-ip: 10.201.0.201
      external-amf-hostname: amf.lab
EOF
```

Next, we create a Juju model to represent the Control Plane, using the cloud `control-plane-cluster`, which was created in the Bootstrap step.

```bash
juju add-model control-plane control-plane-cluster
```

Deploy the full bundle of software for the control plane:
```bash
juju deploy sdcore-control-plane --trust --channel=edge --overlay control-plane-overlay.yaml
```

Lastly, expose the Software as a Service offer for the AMF.  This is only required if the gNB is deployed using a charm and has been designed to consume the AMF offer of the 5g N2 interface from the core.

```bash
juju offer control-plane.amf:fiveg-n2
```

````

````{tab-item} SD-Core User Plane
Create a Juju model to represent the User Plane, using the cloud `user-plane-cluster`, which was created in the Bootstrap step.  The following parameters can be set:

- `access-gateway-ip`: this is the IP address of the gateway that knows how to route traffic from the UPF towards the gNB subnet
- `access-interface`: the name of the macvlan interface on the Kubernetes host cluster to bridge to the `access` subnet
- `access-ip`: the IP address for the UPF to use on the `access` subnet
- `core-gateway-ip`: this is the IP address of the gateway that knows how to route traffic from the UPF towards the internet
- `core-interface`: the name of the macvlan interface on the Kubernetes host cluster to bridge to the `core` subnet
- `core-ip`: the IP address for the UPF to use on the `core` subnet
- `gnb-subnet`: the subnet CIDR where the gNB radios are reachable.  Note: the `access-gateway-ip` is expected to forward the packets from the `access-interface` to the `gnb-subnet`

```bash
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
EOF
```
[`TODO:` explanation of the gNB subnet?  Here we used a new subnet, but nowhere in the doc do we create or explain it as it belongs in a gNB simulator how to]

Next, we create a Juju model to represent the User Plane, using the cloud `user-plane-cluster`, which was created in the Bootstrap step.

```bash
juju add-model user-plane user-plane-cluster
```

Deploy the bundle of software for the user plane:

```bash
juju deploy sdcore-user-plane --trust --channel=edge --overlay upf-overlay.yaml
```
````
`````
