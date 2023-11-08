# 3. Configure SD-Core

Configuring the 5G core network is done making HTTP requests to the `webui` service. Here we will 
create a subscriber, a device group and a network slice.

Run `juju status` and note the IP address next to the `webui` unit.

```console
ubuntu@host:~$ juju status
Model  Controller          Cloud/Region        Version  SLA          Timestamp
core   microk8s-localhost  microk8s/localhost  3.1.2    unsupported  11:55:31-04:00

App                Version  Status   Scale  Charm              Channel        Rev  Address         Exposed  Message
[...]      
webui                       active       1  sdcore-webui       edge             5  10.152.183.128  no       

Unit                  Workload  Agent  Address      Ports  Message
[...]        
webui/0*              active    idle   10.1.19.189 
```

```{note}
Here the address is `10.1.19.189`, yours will likely be different. Make sure to replace it in
the next command.
```

Export your `webui` IP address to a variable:

```console
export WEBUI_IP="10.1.19.189"
```

Create a new subscriber:

```console
curl -v ${WEBUI_IP}:5000/api/subscriber/imsi-208930100007487 \
--header 'Content-Type: text/plain' \
--data '{
    "UeId":"208930100007487",
    "opc":"981d464c7c52eb6e5036234984ad0bcf",
    "key":"5122250214c33e723a5dd523fc145fc0",
    "sequenceNumber":"16f3b3f70fc2"
}'
```

Create a new device group named "cows" that contains the newly created subscriber:

```console
curl -v ${WEBUI_IP}:5000/config/v1/device-group/cows \
--header 'Content-Type: application/json' \
--data '{
    "imsis": [
        "208930100007487"
    ],
    "site-info": "demo",
    "ip-domain-name": "pool1",
    "ip-domain-expanded": {
        "dnn": "internet",
        "ue-ip-pool": "172.250.1.0/16",
        "dns-primary": "8.8.8.8",
        "mtu": 1460,
        "ue-dnn-qos": {
            "dnn-mbr-uplink": 20000000,
            "dnn-mbr-downlink": 200000000,
            "bitrate-unit": "bps",
            "traffic-class": {
                "name": "platinum",
                "arp": 6,
                "pdb": 300,
                "pelr": 6,
                "qci": 8
            }
        }
    }
}'
```

Create a network slice called "default" that contains the "cows" device group:

```console
curl -v ${WEBUI_IP}:5000/config/v1/network-slice/default \
--header 'Content-Type: application/json' \
--data '{
  "slice-id": {
    "sst": "1",
    "sd": "010203"
  },
  "site-device-group": [
    "cows"
  ],
  "site-info": {
    "site-name": "demo",
    "plmn": {
      "mcc": "208",
      "mnc": "93"
    },
    "gNodeBs": [
      {
        "name": "demo-gnb1",
        "tac": 1
      }
    ],
    "upf": {
      "upf-name": "upf-external",
      "upf-port": "8805"
    }
  }
}'
```
