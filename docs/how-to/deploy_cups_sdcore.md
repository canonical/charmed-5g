# Deploy SD-Core with CUPS

This guide covers how to install a multi-node SD-Core with Control Plane and User Plane Separation (CUPS).

## Requirements

- Juju >= 3.1
- A load balancer type Juju controller has been bootstrapped, and is externally reachable
- A Control Plane Kubernetes cluster configured with
  - A load balancer with at least 1 available IP address for the Access and Mobility Management Function (AMF)
- A User Plane Kubernetes cluster configured with
  - A load balancer with at least 1 available IP address for the User Plane Function (UPF)
  - Multus
  - 2 MACVLAN interfaces, one each for access and core networks
- 1 Juju cloud per Kubernetes cluster named `control-plane-cluster` and `user-plane-cluster`

## Guide Sample Values

For the purpose of this guide, the following values will be used.

### Networks
 
 | Name | Subnet | Gateway IP |
 | ---- | ------ | ---------- |
 | access | 10.202.0.0/24 | 10.202.0.1 |
 | core   | 10.203.0.0/24 | 10.203.0.1 |
 | gNB    | 10.204.0.0/24 | 10.204.0.1 |


### Domain Name Server Entries

| Name | IP Address | Usage |
| ---- | ---------- | ----- |
| `upf.core` | 10.201.0.151 | Load balancer IP address for UPF Control Plane |
| `amf.core` | 10.201.0.201 | Load balancer IP address for AMF Control Plane |

### MACVLAN Interfaces

| MACVLAN | Purpose |
|---------|---------|
| `access` | Maps to the `access` subnet |
| `core`   | Maps to the `core` subnet |

### Juju Clouds

| Cloud Name | Purpose |
|------------|---------|
| `control-plane-cluster` | Represents the Kubernetes cluster that executes all control plane functions |
| `user-plane-cluster`    | Represents the Kubernetes cluster that executes all user plane functions |

## Deploy SD-Core Control Plane

Create a Juju overlay file that specifies the:
- AMF host name
- IP address for sharing with the radios

This host name must be resolvable by the gNB and the IP address must be reachable and resolve to the AMF unit.

Example:

```console
cat << EOF > control-plane-overlay.yaml
applications:
  amf:
    options:
      external-amf-ip: 10.201.0.201
      external-amf-hostname: amf.core
EOF
```

Create a Juju model to represent the Control Plane, using the cloud `control-plane-cluster`.

```console
juju add-model control-plane control-plane-cluster
```

Deploy the control plane bundle:
```console
juju deploy sdcore-control-plane --trust --channel=edge --overlay control-plane-overlay.yaml
```

Expose the Software as a Service offer for the AMF.  This is only required if the gNB is deployed using a charm and has been designed to consume the AMF offer of the 5g N2 interface from the core.

```console
juju offer control-plane.amf:fiveg-n2
```

## Deploy SD-Core User Plane

Create a Juju overlay file that specifies the Access and Mobility Management Function (AMF) host name and IP address for sharing with the radios.  This host name must be resolvable by the gNB and the IP address must be reachable and resolve to the AMF unit.

Create a Juju overlay file that specifies the:
- Access gateway IP address
- Access interface MACVLAN name
- Access IP address to use for the UPF
- Core gateway IP address
- Core interface MACVLAN name
- Core IP address to use for the UPF
- gNB subnet that all radios for this UPF are on

Example:

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
EOF
```

Create a Juju model to represent the User Plane, using the cloud `control-plane-cluster`.

```console
juju add-model user-plane user-plane-cluster
```

Deploy user plane bundle:

```console
juju deploy sdcore-user-plane --trust --channel=edge --overlay upf-overlay.yaml
```
