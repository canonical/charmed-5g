# Deploy the SD-CORE UPF

This guide explains how to deploy the SD-CORE UPF charmed operator as a standalone network 
function on Kubernetes.

## Requirements

- Juju >= 3.0
- Kubernetes >= 1.25
- Multus

## Deploy the UPF

Enable Multus on MicroK8s

```bash
sudo microk8s addons repo add community https://github.com/canonical/microk8s-community-addons --reference feat/strict-fix-multus
sudo microk8s enable multus
```

Create a Juju model:

```bash
juju add-model user-plane
```

Deploy the UPF:

```bash
juju deploy sdcore-upf --trust --channel=edge
```

The UPF is ready when the charm is in the `Active/Idle` state.
