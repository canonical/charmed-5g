# Deploy the SD-Core Control Plane

This guide explains how to deploy the SD-Core Control Plane charm bundle.

## Requirements

- Juju >= 3.1
- Kubernetes >= 1.25

## Deploy the SD-Core Control Plane

Create a Juju model:

```bash
juju add-model control-plane
```

Deploy the `sdcore-control-plane` charm bundle:

```bash
juju deploy sdcore-control-plane --trust --channel=edge
```

The 5G control plane is ready when all the charms are in the `Active/Idle` state.
