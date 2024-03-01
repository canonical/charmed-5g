# Deploy SD-Core with Control Plane and User Plane Separation

This guide covers how to install a SD-Core 5G core network with Control Plane and User Plane Separation (CUPS).

## Requirements

- Juju >= 3.4
- A Juju controller has been bootstrapped, and is externally reachable
- A Control Plane Kubernetes cluster configured with
  - 1 available IP address for the Access and Mobility Management Function (AMF)
- A User Plane Kubernetes cluster configured with
  - 1 available IP address for the User Plane Function (UPF)
  - Multus
  - MACVLAN or SR-IOV interfaces for Access and Core networks
- 1 Juju cloud per Kubernetes cluster named `control-plane-cloud` and `user-plane-cloud` respectively

## Deploy SD-Core Control Plane

Create a Juju overlay file.

```console
cat << EOF > control-plane-overlay.yaml
applications:
  amf:
    options:
      external-amf-ip: 10.201.0.201
      external-amf-hostname: amf.core
EOF
```

Create a Juju model to represent the Control Plane.

```console
juju add-model control-plane control-plane-cloud
```

Deploy the control plane bundle.
```console
juju deploy sdcore-control-plane-k8s --trust --channel=beta --overlay control-plane-overlay.yaml
```

Expose the integration offer for the AMF N2 interface.

```console
juju offer control-plane.amf:fiveg-n2
```

## Deploy SD-Core User Plane

Create a Juju overlay file.

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
EOF
```

Create a Juju model.

```console
juju add-model user-plane user-plane-cloud
```

Deploy the user plane bundle.

```console
juju deploy sdcore-user-plane-k8s --trust --channel=beta --overlay upf-overlay.yaml
```
