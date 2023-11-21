# Getting started

In this tutorial, we will deploy and run the SD-Core 5G core network using Juju. We will also
deploy a radio and cellphone simulator to simulate usage of this network.

To complete this tutorial, you will need a machine with the following
requirements:

- A recent `x86_64` CPU (Intel 4ᵗʰ generation or newer, or AMD Ryzen or newer)
- 8GB of RAM
- 50GB of free disk space

## 1. Bootstrap a Juju controller

We will get started by installing MicroK8s on your machine and bootstrapping a Juju controller on 
this MicroK8s instance. 

### Install MicroK8s

From your terminal, install MicroK8s:

```console
sudo snap install microk8s --channel=1.27-strict/stable
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

### Install Juju

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

## 2. Deploy SD-Core

Now we will deploy the `sdcore` charm bundle that contains a complete 5G core network.

### Enable the Multus MicroK8s addon

Add the community repository addon:

```console
sudo microk8s addons repo add community https://github.com/canonical/microk8s-community-addons --reference feat/strict-fix-multus
```

Enable the Multus and MetalLB MicroK8s addons. We must give MetalLB an address
range that has at least 3 IP addresses for Charmed 5G.

```console
sudo microk8s enable multus
sudo microk8s enable metallb:10.0.0.2-10.0.0.4
```

### Deploy the `sdcore` charm bundle

Create a Juju model named `core`:

```console
juju add-model core
```

Deploy the `sdcore-router` operator:

```console
juju deploy sdcore-router router --trust --channel=edge
```

Deploy the `sdcore` charm bundle:

```console
juju deploy sdcore --trust --channel=edge
```

Deploying the core network can take up to 15 minutes. You can validate the status of the
deployment by running `juju status`. The core network is ready when all the charms are in the
`Active/Idle` state. It is normal for `grafana-agent-k8s` to remain in waiting state. Example:

```console
ubuntu@host:~$ juju status
Model  Controller          Cloud/Region        Version  SLA          Timestamp
core   microk8s-localhost  microk8s/localhost  3.1.2    unsupported  11:55:31-04:00

App                Version  Status   Scale  Charm              Channel        Rev  Address         Exposed  Message
amf                         active       1  sdcore-amf         edge             6  10.152.183.187  no
ausf                        active       1  sdcore-ausf        edge             3  10.152.183.205  no
grafana-agent-k8s  0.26.1   waiting      1  grafana-agent-k8s  latest/stable   24  10.152.183.196  no       installing agent
mongodb-k8s                 active       1  mongodb-k8s        5/edge          29  10.152.183.194  no
nrf                         active       1  sdcore-nrf         edge            18  10.152.183.121  no
nssf                        active       1  sdcore-nssf        edge             4  10.152.183.177  no
pcf                         active       1  sdcore-pcf         edge             2  10.152.183.139  no
router                      active       1  sdcore-router      edge             1  10.152.183.250  no
smf                         active       1  sdcore-smf         edge             3  10.152.183.239  no
udm                         active       1  sdcore-udm         edge             2  10.152.183.254  no
udr                         active       1  sdcore-udr         edge             5  10.152.183.200  no
upf                         active       1  sdcore-upf         edge             7  10.152.183.32   no
webui                       active       1  sdcore-webui       edge             5  10.152.183.128  no

Unit                  Workload  Agent  Address      Ports  Message
amf/0*                active    idle   10.1.19.135
ausf/0*               active    idle   10.1.19.136
grafana-agent-k8s/0*  waiting   idle   10.1.19.131         no related Prometheus remote-write
mongodb-k8s/0*        active    idle   10.1.19.139
nrf/0*                active    idle   10.1.19.190
nssf/0*               active    idle   10.1.19.167
pcf/0*                active    idle   10.1.19.130
router/0*             active    idle   10.1.19.132
smf/0*                active    idle   10.1.19.137
udm/0*                active    idle   10.1.19.142
udr/0*                active    idle   10.1.19.154
upf/0*                active    idle   10.1.19.147
webui/0*              active    idle   10.1.19.189
```

## 3. Configure SD-Core

Configuring the 5G core network is done making HTTP requests to the `webui` service. Here we will 
create a subscriber, a device group and a network slice.

Run `juju status` and note the IP address next to the `webui` unit.

```console
ubuntu@host:~$ juju status
Model  Controller          Cloud/Region        Version  SLA          Timestamp
core   microk8s-localhost  microk8s/localhost  3.1.2    unsupported  11:55:31-04:00

App                Version  Status   Scale  Charm              Channel        Rev  Address         Exposed  Message
[...]      
webui                       active       1  sdcore-webui       edge             5  10.152.183.128  no       

Unit                  Workload  Agent  Address      Ports  Message
[...]        
webui/0*              active    idle   10.1.19.189 
```

```{note}
Here the address is `10.1.19.189`, yours will likely be different. Make sure to replace it in
the next command.
```

Export your `webui` IP address to a variable:

```console
export WEBUI_IP="10.1.19.189"
```

Create a new subscriber:

```console
curl -v ${WEBUI_IP}:5000/api/subscriber/imsi-208930100007487 \
--header 'Content-Type: text/plain' \
--data '{
    "UeId":"208930100007487",
    "opc":"981d464c7c52eb6e5036234984ad0bcf",
    "key":"5122250214c33e723a5dd523fc145fc0",
    "sequenceNumber":"16f3b3f70fc2"
}'
```

Create a new device group named "cows" that contains the newly created subscriber:

```console
curl -v ${WEBUI_IP}:5000/config/v1/device-group/cows \
--header 'Content-Type: application/json' \
--data '{
    "imsis": [
        "208930100007487"
    ],
    "site-info": "demo",
    "ip-domain-name": "pool1",
    "ip-domain-expanded": {
        "dnn": "internet",
        "ue-ip-pool": "172.250.1.0/16",
        "dns-primary": "8.8.8.8",
        "mtu": 1460,
        "ue-dnn-qos": {
            "dnn-mbr-uplink": 20000000,
            "dnn-mbr-downlink": 200000000,
            "bitrate-unit": "bps",
            "traffic-class": {
                "name": "platinum",
                "arp": 6,
                "pdb": 300,
                "pelr": 6,
                "qci": 8
            }
        }
    }
}'
```

Create a network slice called "default" that contains the "cows" device group:

```console
curl -v ${WEBUI_IP}:5000/config/v1/network-slice/default \
--header 'Content-Type: application/json' \
--data '{
  "slice-id": {
    "sst": "1",
    "sd": "010203"
  },
  "site-device-group": [
    "cows"
  ],
  "site-info": {
    "site-name": "demo",
    "plmn": {
      "mcc": "208",
      "mnc": "93"
    },
    "gNodeBs": [
      {
        "name": "demo-gnb1",
        "tac": 1
      }
    ],
    "upf": {
      "upf-name": "upf-external",
      "upf-port": "8805"
    }
  }
}'
```

## 5. Run the 5G simulation

### Deploy the simulator

Switch to the `core` Juju model:

```console
juju switch core
```

Deploy the `sdcore-gnbsim` operator

```console
juju deploy sdcore-gnbsim gnbsim --trust --channel=edge
```

Integrate it to the AMF:

```console
juju integrate gnbsim:fiveg-n2 amf:fiveg-n2
```

### Run the simulation

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

## 6. Destroy the environment

Destroy the Juju controller and all its models

```console
juju kill-controller microk8s-localhost
```
