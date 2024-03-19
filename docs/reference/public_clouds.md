# Public Clouds

It is not possible to deploy SD-Core on AWS, Microsoft Azure or GCP using their self-managed Kubernetes services. None of them support SCTP on load balancers, which prevent us from getting all the services up and running.

Microk8s is the preferred Kubernetes distribution for SD-Core.