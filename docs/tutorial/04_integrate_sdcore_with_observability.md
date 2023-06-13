# 5. Integrate SD-Core with Observability

We will integrate the 5G core network with the Canonical Observability Stack (COS).

## Enable the `metallb` MicroK8s addon

Enable the `metallb` addon with a range of 1 IP address:

```console
sudo microk8s enable metallb 10.0.0.2-10.0.0.2
```

## Deploy the `cos-lite` bundle

Create a new Juju model named `cos`:

```console
juju add-model cos
```

Deploy the `cos-lite` charm bundle:

```console
juju deploy cos-lite --trust
```

You can validate the status of the deployment by running `juju status`. COS is ready when all the 
charms are in the `Active/Idle` state.

## Integrate Grafana Agent with Prometheus

First, offer the `receive-remote-write` relation from Prometheus for use in other models:

```console
juju offer cos.prometheus:receive-remote-write
```

Then, consume the relation from the `core` model:

```console
juju switch core
juju consume cos.prometheus
```

Integrate `grafana-agent-k8s` (in the `core` model) with `prometheus` (in the `cos` model):

```console
juju integrate prometheus:receive-remote-write grafana-agent-k8s:send-remote-write
```

## Login to Grafana

Retrieve the Grafana admin password:

```console
juju switch cos
juju run grafana/leader get-admin-password
```

In your browser, navigate to `https://10.0.0.2/cos-grafana`. Login using the "admin" username
and the admin password provided in the last command. 

## Create a dashboard for observing 5G related metrics

1. On the left pane, click on "Dashboard" -> "New" -> "New Dashboard".
2. Create a panel for visualizing UPF upstream throughput:
    1. Click on "Add a new panel"
    2. Select Prometheus as the data source
    3. Create a new query with the following content: `sum(8 * irate(upf_bytes_count{dir="tx",iface="Core"}[2m]))`
    4. Add a title to the panel: "Upstream Bitrate"
    5. Click on "Apply"
3. Create a second panel named "Downstream Bitrate" with the following query: `sum(8 * irate(upf_bytes_count{dir="tx",iface="Access"}[2m]))`.
4. Name the dashboard: "UPF Throughput"
5. Click on "Save Dashboard"

You now have a new dashboard for visualizing your 5G UPF throughput.

```{image} ../images/grafana_upf_dashboard_sim_before.png
:alt: Grafana dashboard
:align: center
```
