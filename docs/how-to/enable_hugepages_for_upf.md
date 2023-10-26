# Enable HugePages in the UPF

## Requirements

- A Kubernetes Host 
  - CPU that supports AVX2 and RDRAND instructions (Intel Haswell, AMD Excavator or equivalent)
  - CPU has the flag "pdpe1gb" which allows to set 1Gi HugePages
  - HugePages is enabled and has enough 2Gi free HugePages
- A Kubernetes cluster with the Multus addon enabled
- Juju >= 3.1/stable

## Deploy the UPF

Deploy the UPF charm in a Juju model named `user-plane` with HugePages enabled.

```bash
juju add-model user-plane
juju deploy sdcore-upf --trust --channel=edge --config enable-hugepages=True
```

## Validate HugePages support

Check the bessd container logs which indicate that the HugePages are reserved.

```bash
$ kubectl logs -f  sdcore-upf-0 -c  bessd -n user-plane  | grep -i "huge-unlink"
2023-10-26T12:33:35.495Z [bessd] I1026 12:33:35.495568    54 dpdk.cc:169] Initializing DPDK EAL with options: ["bessd", "--main-lcore", "127", "--lcore", "127@0-31", "--no-shconf", "--legacy-mem", "--socket-mem", "1024", "--huge-unlink"]
```

Besides, PacketPool is created with name `DpdkPacketPool` in bessd container logs if HugePages is enabled.

```bash
$ kubectl logs -f  sdcore-upf-0 -c  bessd -n user-plane  | grep -i pool
2023-10-26T12:33:35.705Z [bessd] I1026 12:33:35.705854    54 packet_pool.cc:49] Creating DpdkPacketPool for 262144 packets on node 0
```