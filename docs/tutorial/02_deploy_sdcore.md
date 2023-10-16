# 2. Deploy SD-Core

Now we will deploy the `sdcore` charm bundle that contains a complete 5G core network.

## Enable the Multus MicroK8s addon

Add the community repository addon:

```console
sudo microk8s addons repo add community https://github.com/canonical/microk8s-community-addons --reference feat/strict-fix-multus
```

Enable the Multus and MetalLB MicroK8s addons. We must give MetalLB an address
range that has at least 3 IP addresses for Charmed 5G.

```console
sudo microk8s enable multus
sudo microk8s enable metallb:10.0.0.2-10.0.0.5
```

## Deploy the `sdcore` charm bundle

Create a Juju model named `core`:

```console
juju add-model core
```

Deploy the `sdcore-router` operator:

```console
juju deploy sdcore-router router --trust --channel=edge
```

Create a file called `overlay.yaml` in your current working directory and place the following 
content in it:

```yaml
:caption: overlay.yaml
applications:
  traefik-k8s:
    options:
      external_hostname: <your hostname> 
      routing_mode: subdomain
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
core   microk8s-localhost  microk8s/localhost  3.1.6    unsupported  11:34:48+02:00

App                       Version  Status   Scale  Charm                     Channel        Rev  Address         Exposed  Message
amf                                active       1  sdcore-amf                edge            35  10.152.183.24   no       
ausf                               active       1  sdcore-ausf               edge            26  10.152.183.155  no       
grafana-agent-k8s         0.32.1   waiting      1  grafana-agent-k8s         latest/stable   42  10.152.183.225  no       installing agent
mongodb-k8s                        active       1  mongodb-k8s               5/edge          36  10.152.183.243  no       Primary
nms                                active       1  sdcore-nms                edge            12  10.152.183.165  no       
nrf                                active       1  sdcore-nrf                edge            45  10.152.183.142  no       
nssf                               active       1  sdcore-nssf               edge            24  10.152.183.127  no       
pcf                                active       1  sdcore-pcf                edge            19  10.152.183.215  no       
self-signed-certificates           active       1  self-signed-certificates  beta            33  10.152.183.66   no       
smf                                active       1  sdcore-smf                edge            24  10.152.183.91   no       
traefik-k8s               2.9.6    active       1  traefik-k8s               latest/stable  129  10.0.0.3        no       
udm                                active       1  sdcore-udm                edge            22  10.152.183.171  no       
udr                                active       1  sdcore-udr                edge            20  10.152.183.84   no       
upf                                active       1  sdcore-upf                edge            47  10.152.183.240  no       
webui                              active       1  sdcore-webui              edge            15  10.152.183.109  no       

Unit                         Workload  Agent  Address       Ports  Message
amf/0*                       active    idle   10.1.194.235         
ausf/0*                      active    idle   10.1.194.220         
grafana-agent-k8s/0*         blocked   idle   10.1.194.230         grafana-cloud-config: off, logging-consumer: off
mongodb-k8s/0*               active    idle   10.1.194.241         Primary
nms/0*                       active    idle   10.1.194.242         
nrf/0*                       active    idle   10.1.194.195         
nssf/0*                      active    idle   10.1.194.199         
pcf/0*                       active    idle   10.1.194.204         
self-signed-certificates/0*  active    idle   10.1.194.247         
smf/0*                       active    idle   10.1.194.224         
traefik-k8s/0*               active    idle   10.1.194.226         
udm/0*                       active    idle   10.1.194.212         
udr/0*                       active    idle   10.1.194.203         
upf/0*                       active    idle   10.1.194.193         
webui/0*                     active    idle   10.1.194.198
```
