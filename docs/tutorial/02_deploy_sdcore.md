# Deploy SD-Core

Now we will deploy the `sdcore` charm bundle that contains a complete 5G core network.

## Enable the Multus MicroK8s addon

Add the community repository addon:

```console
sudo microk8s addons repo add community https://github.com/canonical/microk8s-community-addons --reference feat/strict-fix-multus
```

Enable the Multus addon:

```console
sudo microk8s enable multus
```

## Deploy the `sdcore` charm bundle

Create a Juju model named `core`:

```console
juju add-model core
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
smf/0*                active    idle   10.1.19.137         
udm/0*                active    idle   10.1.19.142         
udr/0*                active    idle   10.1.19.154         
upf/0*                active    idle   10.1.19.147         
webui/0*              active    idle   10.1.19.189 
```
