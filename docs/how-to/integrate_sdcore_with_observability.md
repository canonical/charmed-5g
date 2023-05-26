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

## Integrate Grafana Agent with Prometheus

We will create a cross model integration between Grafana Agent (in the SD-Core model) and Prometheus (in the `cos` model).

First, offer the `receive-remote-write` relation from Prometheus for use in other models:

```bash
juju offer cos.prometheus:receive-remote-write
```

Then, consume the relation from the SD-Core model:

```bash
juju switch <SD-Core model>
juju consume cos.prometheus
```

Integrate `grafana-agent-k8s` with `prometheus`:

```bash
juju integrate prometheus:receive-remote-write grafana-agent-k8s:send-remote-write
```

You can now see metrics coming from SD-Core in your Grafana dashboard.

```{image} ../images/grafana_upf.png
:alt: Grafana UPF Metrics
:width: 700px
:align: center
```
