# Bootstrap a Juju controller

We will get started by installing MicroK8s on your machine and bootstrapping a Juju controller on 
this MicroK8s instance. 

## Install MicroK8s

From your terminal, install MicroK8s:

```console
sudo snap install microk8s --channel=1.27-strict/stable
```

Enable the following addons:

```console
sudo microk8s enable dns rbac hostpath-storage
```

Enable the multus addon:

```console
sudo microk8s addons repo add community https://github.com/canonical/microk8s-community-addons --reference feat/strict-fix-multus
sudo microk8s enable multus
```

## Install Juju

From your terminal, install Juju.

```console
sudo snap install juju --channel=3.1/stable
```

Bootstrap a Juju controller

```console
juju bootstrap microk8s
```
