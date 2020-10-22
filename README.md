# kubeadm-gcp-experimental

Experimental to deploy k8s cluster on Google Cloud with kubeadm and calico

## Presequite

- Google Cloud Account
- Configured Google Cloud SDK cli

## Cluster Component


![cluster-component](./images/cluster-component.png)

## Steps to setup


## Create Resources


Create VPC Network


```bash
gcloud compute networks create k8s-net --subnet-mode custom
gcloud compute networks subnets create k8s-subnet \
  --network k8s-net \
  --range 10.240.0.0/24
gcloud compute firewall-rules create k8s-allow-internal \
  --allow tcp,udp,icmp,ipip \
  --network k8s-net \
  --source-ranges 10.240.0.0/24
```

Create bastion host, add network tag and allow ssh from ssh to bastion host.

Create etcd host in k8s-network

## References

- <https://docs.projectcalico.org/getting-started/kubernetes/self-managed-public-cloud/gce>