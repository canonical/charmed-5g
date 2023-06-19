# 1. Bootstrap a Juju controller

We will get started by installing MicroK8s on your machine and bootstrapping a Juju controller on 
this MicroK8s instance. 

## Install MicroK8s

From your terminal, install MicroK8s:

```console
sudo snap install microk8s --channel=1.27-strict/stable
```

Enable the following addons:

```console
sudo microk8s enable dns
sudo microk8s enable rbac
sudo microk8s enable hostpath-storage
```

## Install Juju

From your terminal, install Juju.

```console
sudo snap install juju --channel=3.1/stable
```

Bootstrap a Juju controller

```{note}
If you have never had Juju installed on your machine before, prior to bootstrapping 
a Juju controller, from your terminal, create directories required by Juju:
mkdir -p /home/ubuntu/.local/share
This is a workaround to [a Juju bug](https://bugs.launchpad.net/juju/+bug/1988355).
```

```console
juju bootstrap microk8s
```
