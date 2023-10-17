# 3. Configure SD-Core

Configuration of the 5G core network is done using the NMS (Network Management System) graphical
interface. Here we will create a subscriber, a device group and a network slice.

Access the NMS at `https://<model name>-nms.<your hostname>` and click on the `Configure` button
and create a new network with MCC of `208` and MNC of `93`:

```{image} ../images/configure_5g_network.png
:alt: Create 5G network
:align: center
```

Create a new subscriber using following values:

```text
IMSI: 208930100007487
OPC: 981d464c7c52eb6e5036234984ad0bcf
Key: 5122250214c33e723a5dd523fc145fc0
Sequence Number: 16f3b3f70fc2
```

```{image} ../images/create_subscriber.png
:alt: Create subscriber
:align: center
```

Final result should look like below:

```{image} ../images/network_configured.png
:alt: Properly configured 5G network
:align: center
```
