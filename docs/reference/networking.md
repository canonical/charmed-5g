# 5G Network Architecture

5G was built with network functions divided by services which is known as the 5G core Service-Based Architecture (SBA). The 5G core is a network of interconnected services, as illustrated in the figure below.

```{image} ../images/5g_network_architecture.png
:alt: 5G Network Architecture
:height: 600
:align: center
```

The 5G Core Network is composed of various network functions, each serving a unique purpose. These functions communicate internally and externally over well-defined standard interfaces.
The User Plane Function is a critical component of the 5G core network architecture. It oversees the management of user data during the data transmission process. The UPF serves as a connection point between the RAN and the data network.
UPF Interfaces/reference points with employed protocols:
- Access (N3): Interface between the RAN (gNB) and the UPF
- Core (N6): Interface between the Data Network (DN) and the UPF
- K8s LoadBalancer (N4): Interface between the Session Management Function (SMF) and the UPF  


UPF endpoints are given in the below table:

| **UPF PORT NAME** | **PORT NUMBER**      | **PROTOCOL**  |
|:------------------|:---------------------|:--------------|
| PFCP              | 8805                 | UDP           |
| BESSD             | 10514                | TCP           |



AMF endpoints are given in the below table:

| **AMF PORT NAME** | **PORT NUMBER**     | **PROTOCOL** |
|:------------------|:--------------------|:-------------|
| NGAPP             | 38412               | SCTP         |
| SBI               | 29518               | HTTPS        |
| GRPC              | 9000                | TCP          |
                         
