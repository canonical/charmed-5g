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

## Deploy the `cos-configuration-k8s` charm

Deploy the `cos-configuration-k8s` charm with the following SD-Core COS configuration:

```console
juju deploy cos-configuration-k8s \
  --config git_repo=https://github.com/canonical/sdcore-cos-configuration \
  --config git_branch=main \
  --config git_depth=1 \
  --config grafana_dashboards_path=grafana_dashboards/sdcore/
```

Relate it to Grafana:

```console
juju integrate cos-configuration-k8s grafana
```

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
and the admin password provided in the last command. Click on "Dashboards" -> "Browse" and select 
"5G Network Overview".

This dashboard presents an overview of your 5G Network status. Keep this page open, we will
revisit it shortly.

```{image} ../images/grafana_5g_dashboard_sim_before.png
:alt: Grafana dashboard
:align: center
```
