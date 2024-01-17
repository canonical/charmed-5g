# Getting started

In this tutorial, we will deploy and run the SD-Core 5G core network using Juju. We will also
deploy a radio and cellphone simulator to simulate usage of this network.

To complete this tutorial, you will need a machine with the following
requirements:

- A recent `x86_64` CPU (Intel 4ᵗʰ generation or newer, or AMD Ryzen or newer)
- 8GB of RAM
- 50GB of free disk space

## 1. Install MicroK8s

From your terminal, install MicroK8s:

```console
sudo snap install microk8s --channel=1.27-strict/stable
```

Add your user to the `snap_microk8s` group:

```console
sudo usermod -a -G snap_microk8s $USER
newgrp snap_microk8s
```

Add the community repository MicroK8s addon:

```console
sudo microk8s addons repo add community https://github.com/canonical/microk8s-community-addons --reference feat/strict-fix-multus
```

Enable the following MicroK8s addons. We must give MetalLB an address
range that has at least 3 IP addresses for Charmed 5G.

```console
sudo microk8s enable hostpath-storage
sudo microk8s enable multus
sudo microk8s enable metallb:10.0.0.2-10.0.0.4
```

## 2. Bootstrap a Juju controller

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
`mkdir -p /home/ubuntu/.local/share`
```

## 3. Deploy SD-Core


Create a Juju model named `core`:

```console
juju add-model core
```

Deploy the `sdcore-router-k8s` operator:

```console
juju deploy sdcore-router-k8s router --trust --channel=beta
```

Deploy the `sdcore-k8s` charm bundle:

```console
juju deploy sdcore-k8s --trust --channel=beta
```

Deploying the core network can take up to 15 minutes. You can validate the status of the
deployment by running `juju status`. The core network is ready when all the charms are in the
`Active/Idle` state. It is normal for `grafana-agent-k8s` to remain in waiting state. Example:

```console
ubuntu@host:~$ juju status
Model  Controller          Cloud/Region        Version  SLA          Timestamp
core   microk8s-localhost  microk8s/localhost  3.1.6    unsupported  12:58:34-05:00

App                       Version  Status   Scale  Charm                     Channel        Rev  Address         Exposed  Message
amf                                active       1  sdcore-amf-k8s                edge            57  10.152.183.208  no
ausf                               active       1  sdcore-ausf-k8s               edge            40  10.152.183.237  no
gnbsim                             active       1  sdcore-gnbsim-k8s             edge            43  10.152.183.167  no
grafana-agent-k8s         0.32.1   waiting      1  grafana-agent-k8s             latest/stable   44  10.152.183.245  no       installing agent
mongodb-k8s                        active       1  mongodb-k8s                   6/beta          36  10.152.183.156  no       Primary
nms                                active       1  sdcore-nms-k8s                edge            26  10.152.183.121  no
nrf                                active       1  sdcore-nrf-k8s                edge            62  10.152.183.123  no
nssf                               active       1  sdcore-nssf-k8s               edge            37  10.152.183.165  no
pcf                                active       1  sdcore-pcf-k8s                edge            32  10.152.183.205  no
router                             active       1  sdcore-router-k8s             edge            33  10.152.183.49   no
self-signed-certificates           active       1  self-signed-certificates      beta            33  10.152.183.153  no
smf                                active       1  sdcore-smf-k8s                edge            37  10.152.183.147  no
traefik-k8s               2.10.4   active       1  traefik-k8s                   latest/stable  148  10.0.0.3        no
udm                                active       1  sdcore-udm-k8s                edge            35  10.152.183.168  no
udr                                active       1  sdcore-udr-k8s                edge            31  10.152.183.96   no
upf                                active       1  sdcore-upf-k8s                edge            64  10.152.183.126  no
webui                              active       1  sdcore-webui-k8s              edge            23  10.152.183.128  no

Unit                         Workload  Agent  Address      Ports  Message
amf/0*                       active    idle   10.1.182.23
ausf/0*                      active    idle   10.1.182.18
gnbsim/0*                    active    idle   10.1.182.50
grafana-agent-k8s/0*         blocked   idle   10.1.182.51         logging-consumer: off, grafana-cloud-config: off
mongodb-k8s/0*               active    idle   10.1.182.35         Primary
nms/0*                       active    idle   10.1.182.2
nrf/0*                       active    idle   10.1.182.53
nssf/0*                      active    idle   10.1.182.48
pcf/0*                       active    idle   10.1.182.46
router/0*                    active    idle   10.1.182.57
self-signed-certificates/0*  active    idle   10.1.182.56
smf/0*                       active    idle   10.1.182.27
traefik-k8s/0*               active    idle   10.1.182.40
udm/0*                       active    idle   10.1.182.52
udr/0*                       active    idle   10.1.182.39
upf/0*                       active    idle   10.1.182.60
webui/0*                     active    idle   10.1.182.33
```

## 4. Deploy the 5G simulator

Deploy the `sdcore-gnbsim-k8s` operator

```console
juju deploy sdcore-gnbsim-k8s gnbsim --trust --channel=beta
```

Integrate it to the AMF and the NMS:

```console
juju integrate gnbsim:fiveg-n2 amf:fiveg-n2
juju integrate gnbsim:fiveg_gnb_identity nms:fiveg_gnb_identity
```

## 5. Configure the ingress

Configure Traefik to use an external hostname:

```console
juju config traefik-k8s external_hostname=10.0.0.3.nip.io
```

Here, replace `10.0.0.3` with the Application IP address of the `traefik-k8s` application. You can find it by running `juju status traefik-k8s`.

Retrieve the NMS address:

```console
juju run traefik-k8s/0 show-proxied-endpoints
```

The output should be `http://core-nms.10.0.0.3.nip.io/`. Navigate to this address in your browser.


## 6. Configure the 5G core network through the Network Management System

In the Network Management System (NMS), create a network slice with the following attributes:

- Name: `default`
- MCC: `208`
- MNC: `93`
- UPF: `upf-external.core.svc.cluster.local:8805`
- gNodeB: `core-gnbsim-gnbsim`

You should see the following network slice created:

```{image} ../images/nms_network_slice.png
:alt: NMS Network Slice
:align: center
```

Create a subscriber with the following attributes:
- IMSI: `208930100007487`
- OPC: `981d464c7c52eb6e5036234984ad0bcf`
- Key: `5122250214c33e723a5dd523fc145fc0`
- Sequence Number: `16f3b3f70fc2`
- Network Slice: `default`
- Device Group: `default-default`

You should see the following subscriber created:

```{image} ../images/nms_subscriber.png
:alt: NMS Subscriber
:align: center
```

## 7. Run the 5G simulation

Run the simulation

```console
juju run gnbsim/leader start-simulation
```

The simulation executed successfully if you see `success: "true"` as one of the
output messages:

```console
ubuntu@host:~$ juju run gnbsim/leader start-simulation
Running operation 1 with 1 task
  - task 2 on unit-gnbsim-0

Waiting for task 2...
info: run juju debug-log to get more information.
success: "true"
```

## 8. Destroy the environment

Destroy the Juju controller and all its models

```console
juju kill-controller microk8s-localhost
```
