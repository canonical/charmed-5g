# Deploy ONF gNB Simulator

This guide covers how to install and configure the ONF gNB Simulator.

## Requirements

- Juju >= 3.1
- A Juju controller has been bootstrapped
- A Kubernetes cloud has been added to the Juju controller
- Multus has been enabled for the Kubernetes cloud
- MACVLAN interface for gNB radio subnet is available to Kubernetes

## Guide Sample Values

For the purpose of this guide, the following values will be used.

### Networks
 
| Name   | Subnet |
| ------ | ------ |
| access | 10.202.0.0/24 |
| ran    | 10.204.0.0/24 |

## IP Addresses

| Network  | Address | Purpose |
| -------- | ------- | ------- |
| `access` | 10.202.0.1  | IP address of the gateway for this network
| `access` | 10.202.0.10 | IP address of the UPF
| `ran`    | 10.204.0.1  | IP address of the gateway for this network
| `ran`    | 10.204.0.10 | IP address of the gNB simulator

### MACVLAN Interfaces

| MACVLAN | Purpose |
|---------|---------|
| `ran`   | Maps to the `ran` subnet |

### Juju Clouds

| Cloud Name | Purpose |
|------------|---------|
| `gnbsim-cluster` | Represents the Kubernetes cluster where the simulator is to be deployed |

## Deploy gNB Simulator

Create a Juju model to represent the gNB Simulator application, using the cloud `gnbsim-cluster`.

```terminal
juju add-model gnbsim gnbsim-cluster
```

Deploy the `sdcore-gnbsim` operator, providing values for
- GNB Interface MACVLAN name on the kubernetes host
- GNB IP address to use on the GNB MACVLAN interface
- ICMP Packet Destination, which is the target the simulator will attempt to ping
- UPF Gateway IP address on the GNB MACVLAN interface
- UPF IP address of the UPF can be reached on its `access` network `TODO: this should become a relation`

Example:

```terminal
juju deploy sdcore-gnbsim gnbsim --trust --channel=edge \
  --config gnb-interface=ran \
  --config gnb-ip-address=10.204.0.10/24 \
  --config icmp-packet-destination=8.8.8.8 \
  --config upf-gateway=10.204.0.1 \
  --config upf-ip-address=10.202.0.10
```

Integrate the simulator with the offering from the already deployed SD-Core

```terminal
juju consume control-plane.amf
juju integrate gnbsim:fiveg-n2 amf:fiveg-n2
```

