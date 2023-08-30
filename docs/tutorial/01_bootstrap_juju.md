# 1. Bootstrap a Juju controller

We will get started by installing MicroK8s on your machine and bootstrapping a Juju controller on 
this MicroK8s instance. 

## Install MicroK8s

From your terminal, install MicroK8s:

```console
sudo snap install microk8s --channel=1.27-strict/stable
```

```{note}
There is a [bug](https://github.com/canonical/microk8s/issues/4142) in some versions of
MicroK8s that occurs if the system MTU is not the default of 1500.  This will result
in communication errors when deploying the charms.  If your MTU is not 1500, you can
use the following workaround:

1. Note the Calico MTU:
`ip addr s vxlan.calico`

2. Patch Kubernetes to use that:
`microk8s kubectl patch -n kube-system cm calico-config --patch '{"data": {"veth_mtu": "1372"}}'`

3. Reboot
`sudo init 6`
```

Add your user to the `snap_microk8s` group:

```console
sudo usermod -a -G snap_microk8s $USER
newgrp snap_microk8s
```

Enable the `hostpath-storage` MicroK8s addon:

```console
sudo microk8s enable hostpath-storage
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

```{note}
There is a [bug](https://bugs.launchpad.net/juju/+bug/1988355) in Juju that occurs when 
bootstrapping a controller on a new machine. If you encounter it, create the following 
directory:
`mkdir -p $HOME/.local/share`
```
