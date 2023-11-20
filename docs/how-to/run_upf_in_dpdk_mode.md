# Run UPF in DPDK mode

This guide covers how to deploy the User Plane Function (UPF) in DPDK mode using 
the `sdcore-user-plane` Juju bundle.

## Requirements

- A Kubernetes cluster which meets below requirements:
  - host CPU that supports AVX2, RDRAND and PDPE1GB instructions (Intel Haswell, AMD Excavator 
    or equivalent)
  - SR-IOV interfaces for Access and Core networks
  - At least two 1G HugePages available
  - `driverctl` installed
  - LoadBalancer with 1 available address for the UPF
  - Multus CNI enabled
- Juju >= 3.1/stable
- A Juju controller bootstrapped onto the Kubernetes cluster
- Juju model created

## Set up network interfaces

As `root` user, load the `vfio-pci` driver on the Kubernetes host:

```shell
echo "vfio-pci" > /etc/modules-load.d/vfio-pci.conf
modprobe vfio-pci
```

```{note}
Using `vfio-pci`, by default, needs IOMMU to be enabled. In the environments which do not support
IOMMU, `vfio-pci` needs to be loaded with additional module parameter:
`echo "options vfio enable_unsafe_noiommu_mode=1" > /etc/modprobe.d/vfio-noiommu.conf`
```

Get PCI address of `access` and `core` interfaces:

```shell
$ sudo lshw -c network -businfo
Bus info          Device           Class      Description
=========================================================
pci@0000:00:05.0  ens5             network    Elastic Network Adapter (ENA)
pci@0000:00:06.0  ens6             network    Elastic Network Adapter (ENA) # access interface
pci@0000:00:07.0  ens7             network    Elastic Network Adapter (ENA) # core interface
```

Bind `access` and `core` interfaces to the `vfio-pci` driver:

```shell
sudo driverctl set-override 0000:00:06.0 vfio-pci
sudo driverctl set-override 0000:00:07.0 vfio-pci
```

## Configure Kubernetes cluster

Create ConfigMap with configuration for the [SR-IOV Network Device Plugin]:

```shell
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: sriovdp-config
  namespace: kube-config
data:
  config.json: |
    {
      "resourceList": [
        {
          "resourceName": "intel_sriov_vfio_access",
          "selectors": {
            "pciAddresses": ["0000:00:06.0"]
          }
        },
        {
          "resourceName": "intel_sriov_vfio_core",
          "selectors": {
            "pciAddresses": ["0000:00:07.0"]
          }
        }
      ]
    }
EOF
```

Deploy [SR-IOV Network Device Plugin]:

```shell
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/sriov-network-device-plugin/v3.3/deployments/k8s-v1.16/sriovdp-daemonset.yaml
```

Copy the `vfioveth` CNI under `/opt/cni/bin` on Kubernetes host:

```shell
sudo wget -O /opt/cni/bin/vfioveth https://raw.githubusercontent.com/opencord/omec-cni/master/vfioveth
sudo chmod +x /opt/cni/bin/vfioveth
```

## Deploy SD-Core UPF Operator

Create a Juju overlay file.

```console
cat << EOF > upf-overlay.yaml
applications:
  upf:
    options:
      upf-mode: dpdk
      access-ip: 10.202.0.10/24
      access-gateway-ip: 10.202.0.1
      core-ip: 10.203.0.10/24
      core-gateway-ip: 10.203.0.1
      access-interface-mac-address: 00:b0:d0:63:c2:26
      core-interface-mac-address: 00:b0:d0:63:c2:36
      enable-hugepages: True
EOF
```

Deploy the `sdcore-user-plane` bundle.

```console
juju deploy sdcore-user-plane --trust --channel=edge --overlay upf-overlay.yaml
```


[SR-IOV Network Device Plugin]: https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin