# Deploy the SD-CORE User Plane

This guide explains how to deploy the SD-CORE User Plane charm bundle.

## Requirements

- Juju >= 3.0
- Kubernetes >= 1.25
- Multus

## Deploy the SD-CORE User Plane

Enable Multus on MicroK8s

```bash
sudo microk8s addons repo add community https://github.com/canonical/microk8s-community-addons --reference feat/strict-fix-multus
sudo microk8s enable multus
```

Create a Juju model:

```bash
juju add-model user-plane
```

Deploy the `sdcore-user-plane` charm bundle:

```bash
juju deploy sdcore-user-plane --trust --channel=edge
```

The UPF is ready when the charm is in the `Active/Idle` state.
