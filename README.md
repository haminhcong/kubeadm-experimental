# kubeadm-gcp-experimental

Experimental to deploy k8s cluster on Google Cloud with kubeadm and calico

## Presequite

- Google Cloud Account
- Configured Google Cloud SDK cli

## Cluster Component

![cluster-component](./images/cluster-component.png)

## Steps to setup

### Create Resources

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

Allow vm to access internet: <https://cloud.google.com/solutions/building-internet-connectivity-for-private-vms>

Create bastion host `10.240.0.2`, add network tag and allow ssh from ssh to bastion host.

### Setup bastion host

- Install ansible
- Install git
- Clone this repo to bastion host

### Setup etcd

Create etcd-1 VM in k8s-network with 20 GB SSD, 2 CPU 4 GB RAM, IP `10.240.0.3`

```log
wget https://github.com/kubernetes-sigs/etcdadm/releases/download/v0.1.3/etcdadm-linux-amd64
mv etcdadm-linux-amd64 /usr/local/sbin/etcdadm
ETCDCTL_API=3 etcdadm init --version 3.4.13
```

### Setup K8S Masters

Create three K8S Master VMs: `10.240.0.11, 10.240.0.12, 10.240.0.13`

```bash
gcloud compute instances create k8s-controller-1 \
    --async \
    --boot-disk-size 40GB \
    --can-ip-forward \
    --image-family centos-7 \
    --image-project centos-cloud \
    --machine-type e2-medium \
    --no-address \
    --private-network-ip 10.240.0.11 \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet k8s-subnet \
    --zone asia-southeast1-b \
    --tags k8s-cluster,controller

gcloud compute instances create k8s-controller-2 \
    --async \
    --boot-disk-size 40GB \
    --can-ip-forward \
    --image-family centos-7 \
    --image-project centos-cloud \
    --machine-type e2-medium \
    --private-network-ip 10.240.0.12 \
    --no-address \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet k8s-subnet \
    --zone asia-southeast1-b \
    --tags k8s-cluster,controller

gcloud compute instances create k8s-controller-3 \
    --async \
    --boot-disk-size 40GB \
    --can-ip-forward \
    --image-family centos-7 \
    --image-project centos-cloud \
    --machine-type e2-medium \
    --private-network-ip 10.240.0.13 \
    --no-address \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet k8s-subnet \
    --zone asia-southeast1-b \
    --tags k8s-cluster,controller

```

## References

- <https://docs.projectcalico.org/getting-started/kubernetes/self-managed-public-cloud/gce>
- <https://cloud.google.com/solutions/building-internet-connectivity-for-private-vms>