# Deploy SD-Core

`````{tab-set}

````{tab-item} SD-Core

### Requirements
- Juju >= 3.1
- Kubernetes >= 1.25
- A `LoadBalancer` Service for Kubernetes
- Multus

### Deploy

```bash
juju deploy sdcore --trust --channel=edge
```

### Configure

To view all configuration options, please visit the bundle's [Charmhub page](https://charmhub.io/sdcore/).

````

````{tab-item} SD-Core Control Plane

### Requirements
- Juju >= 3.1
- Kubernetes >= 1.25
- A `LoadBalancer` Service for Kubernetes

### Deploy

```bash
juju deploy sdcore-control-plane --trust --channel=edge
```

### Configure

To view all configuration options, please visit the bundle's [Charmhub page](https://charmhub.io/sdcore-control-plane/).

````

````{tab-item} SD-Core User Plane

### Requirements
- Juju >= 3.1
- Kubernetes >= 1.25
- A `LoadBalancer` Service for Kubernetes
- Multus

### Deploy

```bash
juju deploy sdcore-user-plane --trust --channel=edge
```

### Configure

To view all configuration options, please visit the bundle's [Charmhub page](https://charmhub.io/sdcore-user-plane/).

````

`````
