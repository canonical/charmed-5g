# Comparison with the Helm deployment 

Here are multiple reasons to operate SD-Core using Juju.

## Operations 🎛️

- **Model driven operations**: Integrations between the 5G network functions, MongoDB, Observability, TLS certificates providers are all modeled using Juju integrations. This approach makes for simple model driven operations.
- **Cloud deployment**: SD-Core can be deployed on any cloud or on your laptop, all you need is a Kubernetes.
- **Scaling the database**: The MongoDB database can easily scale up and down to accommodate varying subscriber numbers and availability requirements.

## Security 🔒

- **HTTPS by default**: All communications on SBI interfaces between the 5G network functions are encrypted.
- **Lean container images**: Container images are much smaller than the upstream Docker images which makes for a smaller attack surface. Look at the [AMF container image](https://github.com/canonical/sdcore-amf-rock) for example which is 38% smaller. This is achieved using [Rockcraft](https://canonical-rockcraft.readthedocs-hosted.com/en/latest/). 

## Observability 🖥️

- **Metrics**: The AMF, SMF and UPF network functions expose metrics about status, subscriber connectivity and more.
- **Dashboards**: Useful dashboards about UPF throughput and subscriber connectivity provide visual representations of metrics. 
- **Logging**: MongoDB logs are available for troubleshooting and auditing.

To leverage those, [integrate SD-Core with the Observability stack](../how-to/integrate_sdcore_with_observability). 
