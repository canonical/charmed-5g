# Deploy SD-Core with CUPS

This guide covers how to install a multi-node sdcore with separation of user and control plane traffic.

### Requirements

You will need two kubernetes clusters as follows
- `SD-Core Control Plane`, where the full set of control plane functions will be placed (AMF, SMF, etc)
- `SD-Core User Plane`, where the User Plane Function (UPF) will be placed

An additional machine is required for the Juju controller.  While it is possible to install Juju on the same host as the SD-Core Control Plane, it is not recommended to do so.  [`TODO:` note on metallb for juju controller]

Additionally the following subnets must be present:
- `access`, where the UPF listens for incoming sessions from the gNB
- `core`, where the UPF send out traffic from the gNB towards the internet
- `management`, for Juju and control plane traffic.  Must also have access to the internet for downloading container images, charms, or other updates as needed

[`TODO:` Add note about port security somewhere for the networks]

##### Sample Values

For the purpose of this guide, the following values will be used.

###### Subnets

| Name | VLAN | Subnet | Gateway IP |
| ---- | ---- | ------ | ---------- |
| management | 1201 | 10.201.0.0/24 | 10.201.0.1 |
| access     | 1202 | 10.202.0.0/24 | 10.202.0.1 |
| core       | 1203 | 10.203.0.0/24 | 10.203.0.1 |

###### IP Addresses

The only DNS entry that is required is for the AMF so that the gNB can be told where to find it [`TODO:` link to gnb interface reference]

| Name | IP Address | Purpose |
| ---- | ---------- | ------- |
| juju-controller.k8s.priv | 10.201.0.10 | Management address for Juju machine
| control-plane.k8s.priv | 10.201.0.11 | Management address for control plane cluster machine | 
| user-plane.k8s.priv | 10.201.0.12 | Management address for user plane cluster machine | 
| control-plane.juju.priv | 10.201.0.101 | Juju controller load balancer port |
| upf.core.priv | 10.201.0.151 | IP address for UPF to bind to |
| amf.core.priv | 10.201.0.201 | Externally reachable control plane endpoint for the AMF |


`````{tab-set}

````{tab-item} Bootstrap

This section covers how to install Juju and Microk8s to act as the infrastructure for SD-Core, and may be skipped as needed.

----------------------
#### Juju Controller

Machine Minimum Requirements
- Ubuntu 22.04 LTS
- a minimum of `TODO:` 2  cores
- a mimimum of `TODO:` 4  GiB memory
- a minimum of `TODO:` 20  GiB disk space

Begin by installing Microk8s to hold the Juju controller, and configure Metallb to expose 1 IP address for the it.

```bash
sudo snap install microk8s --channel=1.27-strict/stable
sudo microk8s enable hostpath-storage
sudo microk8s enable metallb:10.201.0.101-10.201.0.101
```

Install Juju and bootstrap the controller:

```bash
sudo snap install 
sudo snap install juju --channel=3.1/stable
sudo juju bootstrap microk8s --config controller-service-type=loadbalancer sdcore
```

----------------------
#### SD-Core Control Plane

All commands in this section are run on the SD-Core Control Plane Kubernetes cluster host.

Install Microk8s and configure Metallb to expose 1 IP address for the AMF:

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
#### SD-Core User Plane

All commands in this section are run on the SD-Core Control Plane Kubernetes cluster host.

Install Microk8s, configure Metallb to expose 1 IP address for the UPF, and add the Multus plugin:

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


````



````{tab-item} SD-Core User Plane
`TODO:`
````



````{tab-item} SD-Core Control Plane
`TODO:`
````

`````
