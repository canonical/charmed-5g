# Integrate the SD-Core User Plane with Observability

This guide explains how to Integrate the SD-Core User Plane with the Canonical Observability Stack (COS).

## Requirements

- Juju >= 3.1
- Kubernetes >= 1.25
- Multus
- The `sdcore-user-plane` bundle deployed in the `user-plane` model

## Deploy the `cos-lite` bundle

Deploy the `cos-lite` model in a Juju model named `cos`:

```bash
juju add-model cos 
juju deploy cos-lite --trust
```

## Integrate Grafana Agent with Prometheus

We will create a cross model integration between Grafana Agent (in the `user-plane` model) and Prometheus (in the `cos` model).

First, offer the `receive-remote-write` relation from Prometheus for use in other models:

```bash
juju offer cos.prometheus:receive-remote-write
```

Then, consume the relation from the `user-plane` model:

```bash
juju switch user-plane
juju consume cos.prometheus
```

Integrate `grafana-agent-k8s` with `prometheus`:

```bash
juju integrate prometheus:receive-remote-write grafana-agent-k8s:send-remote-write
```

You can now see metrics coming from the UPF in your Grafana dashboard.

```{image} ../images/grafana_upf.png
:alt: Grafana UPF Metrics
:width: 700px
:align: center
```
