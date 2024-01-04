# Mastering

In this tutorial, we will deploy and run the SD-Core 5G core network following Control and User Plane Separation (CUPS) principles. The radio and cell phone simulator will also be deployed on an isolated cluster.

```{image} ../images/sdcore_networking.png
:alt: Network diagram
:align: center
```

To complete this tutorial, you will need four machines (virtual or physical), with access to the networks as follows.

| Machine | CPUs | RAM | Disk | Networks |
| ------- | ---- | --- | ---- | -------- |
| Control Plane Kubernetes Cluster     | 4 | 8g | 40g | `management` |
| User Plane Kubernetes Cluster        | 4 | 4g | 20g | `management`, `access`, `core` |
| Juju Controller + Kubernetes Cluster | 2 | 8g | 40g | `management` |
| gNB Simulator Kubernetes  Cluster    | 4 | 4g | 20g | `management`, `ran` |


## Sample Values

For the purpose of this guide, the following values will be used.

### Networks

| Name | Subnet | Gateway IP |
| ---- | ------ | ---------- |
| `management` | 10.201.0.0/24 | 10.201.0.1 |
| `access`     | 10.202.0.0/24 | 10.202.0.1 |
| `core`       | 10.203.0.0/24 | 10.203.0.1 |
| `ran`        | 10.204.0.0/24 | 10.204.0.1 |

### IP Addresses

The following IP addresses are used in this tutorial and must be present in the DNS Server that all hosts are using.

| Name | IP Address | Purpose |
| ---- | ---------- | ------- |
| `juju-controller.mgmt` | 10.201.0.10 | Management address for Juju machine |
| `control-plane.mgmt` | 10.201.0.11 | Management address for control plane cluster machine |
| `user-plane.mgmt` | 10.201.0.12 | Management address for user plane cluster machine |
| `gnbsim.mgmt` | 10.201.0.13 | Management address for the gNB Simulator cluster machine |
| `api.juju-controller.mgmt` | 10.201.0.50 | Juju controller address |
| `cos.mgmt` | 10.201.0.51 | Canonical Observability Stack address |
| `amf.mgmt` | 10.201.0.100 | Externally reachable control plane endpoint for the AMF |
| `control-plane-nms.control-plane.mgmt` | 10.201.0.101 | Externally reachable control plane endpoint for the NMS |
| `upf.mgmt` | 10.201.0.200 | Externally reachable control plane endpoint for the UPF |

## 1. Bootstrap all Machines

This section covers how to install Juju and MicroK8s to act as the infrastructure for SD-Core.

### SD-Core Control Plane

All commands in this section are run on the SD-Core Control Plane Kubernetes cluster host.

First, install Microk8s.

```console
sudo snap install microk8s --channel=1.27-strict/stable
sudo microk8s enable hostpath-storage
sudo usermod -a -G snap_microk8s $USER
newgrp snap_microk8s
```

The control plane needs to expose two services: the AMF and the NMS. In this next step, we enable the MetalLB add on in Microk8s, and give it a range of two IP addresses:

```console
sudo microk8s enable metallb:10.201.0.100-10.201.0.101
```

Export the Kubernetes configuration and copy that to the controller:

```console
sudo microk8s.config > control-plane-cluster.yaml
scp control-plane-cluster.yaml juju-controller.mgmt:
```

### SD-Core User Plane

All commands in this section are run on the SD-Core User Plane Kubernetes cluster host.

Install MicroK8s, configure MetalLB to expose 1 IP address for the UPF (`10.201.0.200`), and add the Multus plugin:

```console
sudo snap install microk8s --channel=1.27-strict/stable
sudo microk8s enable hostpath-storage
sudo microk8s enable metallb:10.201.0.200-10.201.0.200
sudo microk8s addons repo add community \
    https://github.com/canonical/microk8s-community-addons \
    --reference feat/strict-fix-multus
sudo microk8s enable multus
sudo usermod -a -G snap_microk8s $USER
newgrp snap_microk8s
```

Export the Kubernetes configuration and copy that to the controller:

```console
sudo microk8s.config > user-plane-cluster.yaml
scp user-plane-cluster.yaml juju-controller.mgmt:
```

In this guide, the following network interfaces are available on the SD-Core User Plane machine:

| Interface Name | Purpose |
|----------------|---------|
| ens3           | internal Kubernetes management interface. This maps to the `management` subnet and must have an IP address |
| ens4           | access interface. This maps to the `access` subnet, and does not have an IP on the host |
| ens5           | core interface. This maps to the `core` subnet, and does not have an IP on the host. Note that internet egress is required here and routing tables must be set to route gNB generated traffic |

Now we create the MACVLAN bridges for ens4 and ens5. These instructions are put into a file that is executed on reboot so the interfaces will come back.

```console
cat << EOF | sudo tee /etc/rc.local
#!/bin/bash

ip link add access link ens4 type macvlan mode bridge
ip link set dev access up
ip link add core link ens5 type macvlan mode bridge
ip link set dev core up
EOF
sudo chmod +x /etc/rc.local
sudo /etc/rc.local
```

### gNB Simulator

All commands in this section are run on the gNB Simulator Kubernetes cluster host.

Install MicroK8s and add the Multus plugin:

```console
sudo snap install microk8s --channel=1.27-strict/stable
sudo microk8s enable hostpath-storage
sudo microk8s addons repo add community \
    https://github.com/canonical/microk8s-community-addons \
    --reference feat/strict-fix-multus
sudo microk8s enable multus
sudo usermod -a -G snap_microk8s $USER
newgrp snap_microk8s
```

Export the Kubernetes configuration and copy that to the controller:

```console
sudo microk8s.config > gnb-cluster.yaml
scp gnb-cluster.yaml juju-controller.mgmt:
```

In this guide, the following network interfaces are available on the gNB Simulator machine:

| Interface Name | Purpose |
|----------------|---------|
| ens3           | internal Kubernetes management interface. This maps to the `management` subnet and must have an IP address |
| ens4           | ran interface. This maps to the `ran` subnet, and does not have an IP on the host |

Now we create the MACVLAN bridges for ens4, and label them accordingly:

```console
cat << EOF | sudo tee /etc/rc.local
#!/bin/bash

ip link add ran link ens4 type macvlan mode bridge
ip link set dev ran up
EOF
sudo chmod +x /etc/rc.local
sudo /etc/rc.local
```

### Juju Controller

Begin by installing MicroK8s to hold the Juju controller, and configure MetalLB to expose one IP address for the controller (`10.201.0.50`), and one for the Canonical Observability Stack (`10.201.0.51)`.

```console
sudo snap install microk8s --channel=1.27-strict/stable
sudo microk8s enable hostpath-storage
sudo microk8s enable metallb:10.201.0.50-10.201.0.51
sudo usermod -a -G snap_microk8s $USER
newgrp snap_microk8s
```

Install Juju and bootstrap the controller to the local Microk8s install as a load balancer service. This will expose the Juju controller on the first allocated MetalLB address.

```console
mkdir -p ~/.local/share/juju
sudo snap install juju --channel=3.1/stable
juju bootstrap microk8s --config controller-service-type=loadbalancer sdcore
```

At this point, the Juju controller is ready to start managing external clouds. Add the Kubernetes clusters representing the user plane, control plane, and gNB simulator to Juju. This is done by using the Kubernetes configuration file generated when setting up the clusters above.

```console
export KUBECONFIG=control-plane-cluster.yaml
juju add-k8s control-plane-cluster --controller sdcore
export KUBECONFIG=user-plane-cluster.yaml
juju add-k8s user-plane-cluster --controller sdcore
export KUBECONFIG=gnb-cluster.yaml
juju add-k8s gnb-cluster --controller sdcore
```

You may now proceed to deploy the SD-Core Control Plane and SD-Core User Plane

## 2. Deploy SD-Core Control Plane

The following steps build on the Juju controller which was bootstrapped and knows how to manage the SD-Core Control Plane Kubernetes cluster.

First, create a Juju overlay file that specifies the Access and Mobility Management Function (AMF) host name and IP address for sharing with the radios. This host name must be resolvable by the gNB and the IP address must be reachable and resolve to the AMF unit. In the bootstrap step, we set the Control Plane MetalLB range to start at `10.201.0.100`, so that is what we use in the configuration.

```console
cat << EOF > control-plane-overlay.yaml
applications:
  amf:
    options:
      external-amf-ip: 10.201.0.100
      external-amf-hostname: amf.mgmt
EOF
```

Next, we create a Juju model to represent the Control Plane, using the cloud `control-plane-cluster`, which was created in the Bootstrap step.

```console
juju add-model control-plane control-plane-cluster
```

Deploy the control plane Juju bundle:
```console
juju deploy sdcore-control-plane-k8s --trust --channel=beta --overlay control-plane-overlay.yaml
```

Expose the Software as a Service offer for the AMF.

```console
juju offer control-plane.amf:fiveg-n2
```

Configure the NMS to be served as `control-plane.mgmt`:

```console
juju config traefik-k8s external_hostname=control-plane.mgmt
```

### Checkpoint

You should be able to see the AMF External load balancer service in Kubernetes. Log into the control plane cluster and execute the following command:

```console
microk8s.kubectl get services -A | grep LoadBalancer
```

This will show output similar to the following, indicating:
- The AMF is exposed on 10.201.0.100 SCTP port 38412
- The NMS is exposed on 10.201.0.101 TCP ports 80 and 443

These IP addresses came from MetalLB and were configured in the bootstrap step.

```
control-plane    amf-external  LoadBalancer  10.152.183.179  10.201.0.100   38412:30408/SCTP
control-plane    traefik-k8s   LoadBalancer  10.152.183.28   10.201.0.101   80:32349/TCP,443:31925/TCP
```

## 3. Deploy SD-Core User Plane

Create a Juju model to represent the User Plane, using the cloud `user-plane-cluster`, which was created in the Bootstrap step. The following parameters can be set:

- `access-gateway-ip`: this is the IP address of the gateway that knows how to route traffic from the UPF towards the gNB subnet
- `access-interface`: the name of the MACVLAN interface on the Kubernetes host cluster to bridge to the `access` subnet
- `access-ip`: the IP address for the UPF to use on the `access` subnet
- `core-gateway-ip`: this is the IP address of the gateway that knows how to route traffic from the UPF towards the internet
- `core-interface`: the name of the MACVLAN interface on the Kubernetes host cluster to bridge to the `core` subnet
- `core-ip`: the IP address for the UPF to use on the `core` subnet
- `gnb-subnet`: the subnet CIDR where the gNB radios are reachable.

```{note}
The `access-gateway-ip` is expected to forward the packets from the `access-interface` to the `gnb-subnet`
```

```console
cat << EOF > upf-overlay.yaml
applications:
  upf:
    options:
      access-gateway-ip: 10.202.0.1
      access-interface: access
      access-ip: 10.202.0.10/24
      core-gateway-ip: 10.203.0.1
      core-interface: core
      core-ip: 10.203.0.10/24
      gnb-subnet: 10.204.0.0/24
      external-upf-hostname: upf.mgmt
EOF
```

Next, we create a Juju model to represent the User Plane, using the cloud `user-plane-cluster`, which was created in the Bootstrap step.

```console
juju add-model user-plane user-plane-cluster
```

Deploy user plane bundle:

```console
juju deploy sdcore-user-plane-k8s --trust --channel=beta --overlay upf-overlay.yaml
```

Now expose the UPF service offering with Juju.

```console
juju offer user-plane.upf:fiveg_n4
```

### Checkpoint

You should be able to see the UPF External load balancer service in Kubernetes. Log into the user plane cluster and execute the following command:

```console
microk8s.kubectl get services -A | grep LoadBalancer
```

This should produce output similar to the following indicating that the PFCP agent of the UPF is exposed on 10.201.0.200 UDP port 8805
```
user-plane  upf-external  LoadBalancer  10.152.183.126  10.201.0.200  8805:31101/UDP
```

## 4. Deploy the gNB Simulator

Create a new Juju model named `gnbsim` on the Juju controller cloud:

```console
juju add-model gnbsim gnb-cluster
```

Deploy the simulator to the gnbsim cluster. The simulator needs to know the following:

- `gnb-interface`: the name of the MACVLAN interface to use on the host
- `gnb-ip-address`: the IP address to use on the gnb interface
- `icmp-packet-destination`: the target IP address to ping. If there is no egress to the internet on your core network, any IP that is reachable from the UPF should work.
- `upf-gateway`: the IP address of the gateway between the RAN and Access networks
- `upf-subnet`: subnet where the UPFs are located (also called Access network)

```console
juju deploy sdcore-gnbsim-k8s gnbsim --channel=beta \
--config gnb-interface=ran \
--config gnb-ip-address=10.204.0.10/24 \
--config icmp-packet-destination=8.8.8.8 \
--config upf-gateway=10.204.0.1 \
--config upf-subnet=10.202.0.0/16
```

Now we use the integration between the gnbsim operator and the AMF operator to exchange the AMF location information.

```console
juju consume control-plane.amf
juju integrate gnbsim:fiveg-n2 amf:fiveg-n2
```

And present the offering of the gNB identity for integrating with the NMS.

```console
juju offer gnbsim.gnbsim:fiveg_gnb_identity
```

## 5. Configure SD-Core

Still on the Juju controller, switch to the control-plane model.

```console
juju switch control-plane
```

Relate the UPF to the NMS for exchanging identity information.

```console
juju consume user-plane.upf
juju integrate upf:fiveg_n4 nms:fiveg_n4
juju consume gnbsim.gnbsim
juju integrate gnbsim:fiveg_gnb_identity nms:fiveg_gnb_identity
```

The next steps are performed using a web browser to navigate to the NMS at http://control-plane-nms.control-plane.mgmt/

```{note}
The computer that is running the web browser must also use the same DNS server as the rest of the environment.
```

In the Network Management System (NMS), create a network slice with the following attributes:

- Name: `Tutorial`
- MCC: `208`
- MNC: `93`
- UPF: `upf.mgmt:8805`
- gNodeB: `gnbsim-gnbsim-gnbsim (tac:1)`

You should see the following network slice created. Note the device group has been expanded to show the default group that is created in the slice for you.

```{image} ../images/nms_tutorial_network_slice_with_device_group.png
:alt: NMS Network Slice
:align: center
```

We will now add a subscriber with the IMSI that was provided to the gNB simulator. Navigate to Subscribers and click on Create. Fill in the following:

- IMSI: `208930100007487`
- OPC: `981d464c7c52eb6e5036234984ad0bcf`
- Key: `5122250214c33e723a5dd523fc145fc0`
- Sequence Number: `16f3b3f70fc2`
- Network Slice: `Tutorial`
- Device Group: `Tutorial-default`

## 6. Integrate SD-Core with Observability

We will integrate the 5G core network with the Canonical Observability Stack (COS). All commands are to be executed on the Juju controller node.

### Deploy the `cos-lite` bundle

Create a new Juju model named `cos` on the Juju controller cloud:

```console
juju add-model cos microk8s
```

Deploy the `cos-lite` charm bundle. As this is being deployed to the Juju controller's Kubernetes, it will use the second MetalLB IP address (`10.201.0.51`) as its external IP.

```console
juju deploy cos-lite --trust
```

You can validate the status of the deployment by running `juju status`. COS is ready when all the charms are in the `Active/Idle` state.

### Deploy the `cos-configuration-k8s` charm

Deploy the `cos-configuration-k8s` charm with the following SD-Core COS configuration:

```console
juju deploy cos-configuration-k8s \
  --config git_repo=https://github.com/canonical/sdcore-cos-configuration \
  --config git_branch=main \
  --config git_depth=1 \
  --config grafana_dashboards_path=grafana_dashboards/sdcore/
```

Integrate it to Grafana:

```console
juju integrate cos-configuration-k8s grafana
```

### Integrate Grafana Agent with Prometheus

First, offer the following integrations from Prometheus and Loki for use in other models:

```console
juju offer cos.prometheus:receive-remote-write
juju offer cos.loki:logging
```

Then, consume the integrations from the `control-plane` model:

```console
juju switch control-plane
juju consume cos.prometheus
juju consume cos.loki
```

Integrate `grafana-agent-k8s` (in the `control-plane` model) with `prometheus` and `loki` (in the `cos` model):

```console
juju integrate prometheus:receive-remote-write grafana-agent-k8s:send-remote-write
juju integrate loki:logging grafana-agent-k8s:logging-consumer
```

Now, do the same for the `user-plane` model:

```console
juju switch user-plane
juju consume cos.prometheus
juju consume cos.loki
juju integrate prometheus:receive-remote-write grafana-agent-k8s:send-remote-write
juju integrate loki:logging grafana-agent-k8s:logging-consumer
```

### Login to Grafana

Retrieve the Grafana admin password:

```console
juju switch cos
juju run grafana/leader get-admin-password
```

This produces output similar to the following:

```console
Running operation 1 with 1 task
  - task 2 on unit-grafana-0

Waiting for task 2...
admin-password: c72uEq8FyGRo
url: http://10.201.0.51/cos-grafana

```

## Checkpoint

In your browser, navigate to the URL from the output (`https://10.201.0.51/cos-grafana`). Login using the "admin" username and the admin password provided in the last command. Click on "Dashboards" -> "Browse" and select "5G Network Overview".

This dashboard presents an overview of your 5G Network status. Keep this page open, we will revisit it shortly.

```{image} ../images/grafana_5g_dashboard_sim_before.png
:alt: Initial Grafana dashboard showing UPF status
:align: center
```

## 7. Run the 5G simulation

On the juju controller, switch to the gnbsim model.
```console
juju switch gnbsim
```

Start the simulation.

```console
juju run gnbsim/leader start-simulation
```

The simulation executed successfully if you see `success: "true"` as one of the
output messages:

```console
ubuntu@juju-controller:~$ juju run gnbsim/leader start-simulation
Running operation 1 with 1 task
  - task 2 on unit-gnbsim-0

Waiting for task 2...
info: run juju debug-log to get more information.
success: "true"
```

## Checkpoint

### gNB Simulation Logs

Let's take a look at the juju debug-log now by running the following command:

```console
juju debug-log --no-tail
```

This will emit the full log of the simulation starting with the following message:

```console
unit-gnbsim-0: 16:43:50 INFO unit.gnbsim/0.juju-log gnbsim simulation output:
```

As there is a lot of output, we can better understand if we filter by specific elements. For example, let's take a look at the control plane transport of the log. To do that, we search for `ControlPlaneTransport` in the Juju debug-log. This shows the simulator locating the AMF and exchanging data with it.

```console
$ juju debug-log | grep ControlPlaneTransport
2023-11-30T16:43:40Z [TRAC][GNBSIM][GNodeB][ControlPlaneTransport] Connecting to AMF
2023-11-30T16:43:40Z [INFO][GNBSIM][GNodeB][ControlPlaneTransport] Connected to AMF, AMF IP: 10.201.0.100 AMF Port: 38412
...
```

We can do the same for the user plane transport to see it starts on the RAN network with IP address `10.204.0.10` as we requested, and it is communicating with our UPF at `10.202.0.10` as expected.

To follow the UE itself, we can filter by the IMSI.

```console
juju debug-log | grep imsi-208930100007487
```

### Control Plane Logs

You may view the control plane logs by logging into the control plane cluster and using Kubernetes commands as follows:

```bash
kubectl logs -n control-plane -c amf amf-0 --tail 70
kubectl logs -n control-plane -c ausf ausf-0 --tail 70
kubectl logs -n control-plane -c nrf nrf-0 --tail 70
kubectl logs -n control-plane -c nssf nssf-0 --tail 70
kubectl logs -n control-plane -c pcf pcf-0 --tail 70
kubectl logs -n control-plane -c smf smf-0 --tail 70
kubectl logs -n control-plane -c udm udm-0 --tail 70
kubectl logs -n control-plane -c udr udr-0 --tail 70
```

### Grafana Metrics

You can also revisit the Grafana dashboard to view the metrics for the test run. You can see the IMSI is connected and has received an IP address. There is now one active PDU session, and the ping test throughput can be seen in the graphs.

```{image} ../images/grafana_5g_dashboard_sim_after.png
:alt: Grafana dashboard showing throughput metrics
:align: center
```

## 8. Review

We have deployed 4 Kubernetes clusters, bootstrapped a Juju controller to manage them all, and deployed portions of the Charmed 5g SD-Core software according to CUPS principles. You know have 5 Juju models as follows

- `control-plane` where all the control functions are deployed
- `controller` where Juju manages state of the models
- `cos` where the Canonical Observability Stack is deployed
- `gnbsim` where the gNB simulator is deployed
- `user-plane` where all the user plane function is deployed

You have learned how to:

- view the logs for the various functions
- manage the integrations between deployed functions
- run a simulation testing data flow through the 5g core
- view the metrics produced by the 5g core

## 9. Cleaning up

Juju makes it simple to cleanly remove all the deployed applications by simply removing the model itself. To completely remove all deployments, use the following:

```bash
juju destroy-controller --destroy-all-models sdcore
```

You can now proceed to remove Juju itself on the `juju-controller`:

```console
sudo snap remove juju
```

MicroK8s can also be removed from each cluster as follows:

```console
sudo snap remove microk8s
```

Finally you may wish to reboot the servers to ensure no residual network configurations remain.
