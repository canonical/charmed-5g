# Integrate SD-Core with Observability

This guide explains how to Integrate the SD-Core with the Canonical Observability Stack (COS).

## Requirements

- Juju >= 3.1
- Kubernetes >= 1.25
- Multus
- One of the `sdcore` bundles deployed in a Juju model

## Deploy the `cos-lite` bundle

Deploy the `cos-lite` charm bundle in a Juju model named `cos`:

```bash
juju add-model cos 
juju deploy cos-lite --trust
```

## Deploy the `cos-configuration-k8s` charm

Deploy the `cos-configuration-k8s` charm with the following SD-Core COS configuration:

```console
juju deploy cos-configuration-k8s \
  --config git_repo=https://github.com/canonical/sdcore-cos-configuration \
  --config git_branch=main \
  --config git_depth=1 \
  --config grafana_dashboards_path=grafana_dashboards/sdcore/
```

Integrate it with Grafana:

```console
juju integrate cos-configuration-k8s grafana
```

## Integrate Grafana Agent with Prometheus

We will create a cross model integration between Grafana Agent (in the SD-Core model) and Prometheus (in the `cos` model).

First, offer the following integrations from Prometheus and Loki for use in other models:

```bash
juju offer cos.prometheus:receive-remote-write
juju offer cos.loki:logging
```

Then, consume the integrations from the SD-Core model:

```bash
juju switch <SD-Core model>
juju consume cos.prometheus
juju consume cos.loki
```

Integrate `grafana-agent-k8s` with `prometheus` and `loki`:

```bash
juju integrate prometheus:receive-remote-write grafana-agent-k8s:send-remote-write
juju integrate loki:logging grafana-agent-k8s:logging-consumer
```

You can now see metrics and logs coming from SD-Core in your Grafana dashboard.

```{image} ../images/grafana_5g_dashboard_sim_after.png
:alt: Grafana dashboard
:align: center
```
