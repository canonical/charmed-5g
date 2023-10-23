# Deploy SD-Core Standalone

This guide covers how to install a standalone SD-Core 5G core network, suitable for lab or proof of concept purposes.

## Requirements

You will need a Kubernetes cluster installed and configured with Multus.

- Juju >= 3.1
- Kubernetes >= 1.25
- A `LoadBalancer` Service for Kubernetes
- Multus

## Deploy

```bash
<<<<<<< HEAD
juju deploy sdcore-k8s --trust --channel=edge
=======
juju deploy sdcore --trust --channel=edge
>>>>>>> a7cf061 (Initial draft)
```

## Configure

<<<<<<< HEAD
To view all configuration options, please visit the bundle's [Charmhub page](https://charmhub.io/sdcore-k8s/).
=======
To view all configuration options, please visit the bundle's [Charmhub page](https://charmhub.io/sdcore/).
>>>>>>> a7cf061 (Initial draft)
