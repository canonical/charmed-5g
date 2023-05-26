# Integrate with Observability

We will integrate the 5G core network with the Canonical Observability Stack (COS).

## Deploy the `cos-lite` bundle

Create a new juju model named `cos`:

```console
juju add-model cos 
```

Deploy the `cos-lite` charm bundle:

```console
juju deploy cos-lite --trust
```

## Integrate Grafana Agent with Prometheus

First, offer the `receive-remote-write` relation from Prometheus for use in other models:

```bash
juju offer cos.prometheus:receive-remote-write
```

Then, consume the relation from the `core` model:

```bash
juju switch core
juju consume cos.prometheus
```

Integrate `grafana-agent-k8s` with `prometheus`:

```bash
juju integrate prometheus:receive-remote-write grafana-agent-k8s:send-remote-write
```

## Explore metrics

You can now see metrics coming from the UPF in your Grafana dashboard.

```{image} ../images/grafana_upf.png
:alt: Grafana UPF Metrics
:width: 700px
:align: center
```
