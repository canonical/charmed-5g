# 4. Integrate SD-Core with Observability

We will integrate the 5G core network with the Canonical Observability Stack (COS).

## Enable the `metallb` MicroK8s addon

Enable the `metallb` addon with a range of several addresses:

```console
sudo microk8s enable metallb 10.0.0.2-10.0.0.10
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

Integrate it to Grafana:

```console
juju integrate cos-configuration-k8s grafana
```

## Integrate Grafana Agent with Prometheus

First, offer the following integrations from Prometheus and Loki for use in other models:

```console
juju offer cos.prometheus:receive-remote-write
juju offer cos.loki:logging
```

Then, consume the integrations from the `core` model:

```console
juju switch core
juju consume cos.prometheus
juju consume cos.loki
```

Integrate `grafana-agent-k8s` (in the `core` model) with `prometheus` and `loki` (in the `cos` model):

```console
juju integrate prometheus:receive-remote-write grafana-agent-k8s:send-remote-write
juju integrate loki:logging grafana-agent-k8s:logging-consumer
```

## Login to Grafana

Retrieve the Grafana admin password:

```console
juju switch cos
juju run grafana/leader get-admin-password
```

Next, find the IP address of your instance of Traefik, where we will be able to access Grafana, by
inspecting the output of `juju status traefik`. The address will appear under the *App* section, and
it should be in the range we provided to MetalLB. In this case it is `10.0.0.4`

```console
$ juju status traefik
Model  Controller          Cloud/Region        Version  SLA          Timestamp
cos    microk8s-localhost  microk8s/localhost  3.1.5    unsupported  15:34:30-03:00

App      Version  Status  Scale  Charm        Channel  Rev  Address   Exposed  Message
traefik  2.9.6    active      1  traefik-k8s  stable   129  10.0.0.4  no

Unit        Workload  Agent  Address       Ports  Message
traefik/0*  active    idle   10.1.103.186

Offer       Application  Charm           Rev  Connected  Endpoint              Interface                Role
loki        loki         loki-k8s        91   1/1        logging               loki_push_api            provider
prometheus  prometheus   prometheus-k8s  129  1/1        receive-remote-write  prometheus_remote_write  provider
```

In your browser, navigate to `https://<traefik-address>/cos-grafana`. Login using the "admin" username
and the admin password provided in the last command. Click on "Dashboards" -> "Browse" and select
"5G Network Overview".

This dashboard presents an overview of your 5G Network status. Keep this page open, we will
revisit it shortly.

```{image} ../images/grafana_5g_dashboard_sim_before.png
:alt: Grafana dashboard
:align: center
```
