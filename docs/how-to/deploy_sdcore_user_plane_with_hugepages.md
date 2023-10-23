# Deploy SD-Core User Plane with HugePages

This guide covers how to deploy the User Plane Function (UPF) with huge pages using 
the `sdcore-user-plane` Juju bundle.

## Requirements

- A Kubernetes host which meets the following requirements: 
  - CPU that supports AVX2, RDRAND and PDPE1GB instructions (Intel Haswell, AMD Excavator or equivalent)
  - LoadBalancer with 1 available address for the UPF
  - Multus CNI enabled
- Juju >= 3.1/stable
- A Juju controller bootstrapped onto the Kubernetes host
- Juju model created, named `user-plane`

## Enable HugePages on the host
As `root` user, edit the bootloader configuration of the OS (`/etc/default/grub`) appending the following command to the
kernel command-line parameters (`GRUB_CMDLINE_LINUX_DEFAULT`):
```console
default_hugepagesz=1G
```
Then update the bootloader:
```shell
update-grub
```

Allocate 2 HugePages of 1Gi, for a total of 2Gi HugePages, and make it persistent:
```shell
echo "vm.nr_hugepages=2" | tee /etc/sysctl.d/hugepages.conf
```
Then reboot the host.

## Deploy SD-Core User Plane

Create a Juju overlay file.

```console
cat << EOF > upf-overlay.yaml
applications:
  upf:
    options:
      enable-hugepages: True
EOF
```

<<<<<<< HEAD
Deploy the `sdcore-user-plane-k8s` bundle.

```console
juju deploy sdcore-user-plane-k8s --trust --channel=edge --overlay upf-overlay.yaml
=======
Deploy the `sdcore-user-plane` bundle.

```console
juju deploy sdcore-user-plane --trust --channel=edge --overlay upf-overlay.yaml
>>>>>>> a7cf061 (Initial draft)
```

## Validate HugePages support

Check the `bessd` container logs which indicate that the HugePages are reserved.

```console
$ kubectl logs -f  sdcore-upf-0 -c  bessd -n user-plane | grep -i "huge-unlink"
2023-10-26T12:33:35.495Z [bessd] I1026 12:33:35.495568    54 dpdk.cc:169] Initializing DPDK EAL with options: ["bessd", "--main-lcore", "127", "--lcore", "127@0-31", "--no-shconf", "--legacy-mem", "--socket-mem", "1024", "--huge-unlink"]
```

Besides, PacketPool is created with name `DpdkPacketPool` in bessd container logs if HugePages is enabled.

```console
$ kubectl logs -f  sdcore-upf-0 -c  bessd -n user-plane  | grep -i pool
2023-10-26T12:33:35.705Z [bessd] I1026 12:33:35.705854    54 packet_pool.cc:49] Creating DpdkPacketPool for 262144 packets on node 0
<<<<<<< HEAD
```
=======
```
>>>>>>> a7cf061 (Initial draft)
