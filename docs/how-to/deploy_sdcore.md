# Deploy SD-Core

This guide covers how to install a single node sdcore, suitable for lab or proof of concept purposes.

## Requirements

You will need a Kubernetes cluster installed and configured with Multus.

- Juju >= 3.1
- Kubernetes >= 1.25
- A `LoadBalancer` Service for Kubernetes
- Multus

## Deploy

```bash
juju deploy sdcore --trust --channel=edge
```

## Configure

To view all configuration options, please visit the bundle's [Charmhub page](https://charmhub.io/sdcore/).

