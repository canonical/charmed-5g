# Deploy SD-Core

This guide explains how to deploy the SD-Core charm bundle.

## Requirements

- Juju >= 3.1
- Kubernetes >= 1.25
- Multus

## Deploy SD-Core

Enable Multus on MicroK8s

```bash
sudo microk8s addons repo add community https://github.com/canonical/microk8s-community-addons --reference feat/strict-fix-multus
sudo microk8s enable multus
```

Create a Juju model:

```bash
juju add-model sdcore
```

Deploy the `sdcore` charm bundle:

```bash
juju deploy sdcore --trust --channel=edge
```

The 5G core is ready when all the charms are in the `Active/Idle` state.
