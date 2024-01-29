# Deploy SD-Core Standalone (on AWS)

This guide covers how to install a standalone SD-Core 5G core network in AWS.

## Requirements

- An AWS account
- [AWS CLI](https://aws.amazon.com/cli/)
- [EKSCTL CLI](https://eksctl.io/)
- An AWS Route 53 hosted zone

## 1. Create an EKS cluster

Create the Kubernetes cluster:

```bash
eksctl create cluster --name demo --region us-east-2 --node-type t2.xlarge --with-oidc
```

Create the IAM service account:

```bash
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster demo \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --role-only \
  --role-name AmazonEKS_EBS_CSI_DriverRole
```

Get your user ID:

```bash
username=$(aws sts get-caller-identity | jq -r '.Account')
```

Create the EBS CSI driver:
```bash
eksctl create addon --name aws-ebs-csi-driver --cluster demo --service-account-role-arn arn:aws:iam::$username:role/AmazonEKS_EBS_CSI_DriverRole
```

Install Multus:

```bash
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset-thick.yml
```

## 2. Bootstrap Juju

Add the Kubernetes cloud:

```bash
/snap/juju/current/bin/juju add-k8s eks-demo --client --controller aws-us-east-2
```

Bootstrap a Juju controller:

```bash
juju bootstrap eks-firecell01
```

Create a Juju model named `core`:

```bash
juju add-model core
```

## 3. Deploy SD-Core

```bash
juju deploy sdcore-k8s --trust --channel=beta
```

## 4. Configure the Ingress

```bash
juju config traefik-k8s external_hostname=<Your hostname>
```

Create a file `dns.json` with the following content:

```bash
{
  "Comment": "Creating Alias resource record sets in Route 53",
  "Changes": [
    {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "core-nms.<Your hostname>",
        "Type": "CNAME",
	    "TTL": 300,
        "ResourceRecords": [
          {
            "Value": "<LB Address for the Traefik service>"
          }
        ]
      }
    }
  ]
}
```

Create the CNAME records in Route53:

```bash
aws route53 change-resource-record-sets --hosted-zone-id <Your hosted Zone ID> --change-batch  file://route53.json
```

Open a browser and navigate to `https://core-nms.<Your hostname>`. You should see the SD-Core NMS login page.
