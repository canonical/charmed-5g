# Deploy ONF gNB Simulator

This guide covers how to install and configure the ONF gNB Simulator.

## Requirements

- Juju >= 3.1
- A Juju controller has been bootstrapped
- A Kubernetes cloud called `gnbsim-cloud` has been added to the Juju controller 
- Multus has been enabled for the Kubernetes cloud
- MACVLAN interface for gNB radio subnet is available to Kubernetes

## Deploy gNB Simulator

Create a Juju model to represent the gNB Simulator application.

```console
juju add-model gnbsim gnbsim-cloud
```

Deploy the `sdcore-gnbsim` operator charm.

```console
juju deploy sdcore-gnbsim gnbsim --trust --channel=edge \
  --config gnb-interface=ran \
  --config gnb-ip-address=10.204.0.10/24 \
  --config icmp-packet-destination=8.8.8.8 \
  --config upf-gateway=10.204.0.1 \
  --config upf-ip-address=10.202.0.10
```

Integrate the simulator with the offering from the already deployed SD-Core

```console
juju consume control-plane.amf
juju integrate gnbsim:fiveg-n2 amf:fiveg-n2
```