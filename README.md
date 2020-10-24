# kubeadm-gcp-experimental

Experimental to deploy k8s cluster on Google Cloud with kubeadm and calico

## Cluster Configuration

- Region: asia-southeast1-b

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

Setup docker on 3 K8S Master VMs

```bash
ansible-playbook -i inventory.ini site.yml --tags "docker" -e ansible_ssh_user=centos --key-file "/PATH_TO_GOOGLE_CLOUD_VM_KEY" --tags "install_docker"
```

### Setup K8S API LB

Create firewall rule:

```bash
gcloud compute firewall-rules create fw-allow-health-check \
    --network=k8s-net \
    --action=allow \
    --direction=ingress \
    --source-ranges=130.211.0.0/22,35.191.0.0/16 \
    --rules=tcp,udp,icmp
```

Create health check

```bash
# just for test
gcloud compute health-checks create http hc-http-80 \
    --region=asia-southeast1 \
    --port=80
## for K8S API

gcloud compute health-checks create tcp hc-tcp-6443 \
    --region=asia-southeast1 \
    --port=6443
```

Create `k8s-master-group` instance group and add three k8s master VMs to this
instance group:

```bash
gcloud compute instance-groups unmanaged create k8s-master-group \
    --zone=asia-southeast1-b
gcloud compute instance-groups unmanaged add-instances k8s-master-group \
    --zone=asia-southeast1-b \
    --instances=k8s-controller-1,k8s-controller-2,k8s-controller-3
```

Create backend-service

```bash
gcloud compute backend-services create be-k8s-master-nginx \
    --load-balancing-scheme=internal \
    --protocol=tcp \
    --region=asia-southeast1 \
    --health-checks=hc-http-80 \
    --health-checks-region=asia-southeast1
gcloud compute backend-services create be-k8s-master-k8s-api \
    --load-balancing-scheme=internal \
    --protocol=tcp \
    --region=asia-southeast1 \
    --health-checks=hc-tcp-6443 \
    --health-checks-region=asia-southeast1
```

Add k8s master instance group to backend service

```bash
gcloud compute backend-services add-backend be-k8s-master-nginx \
    --region=asia-southeast1 \
    --instance-group=k8s-master-group \
    --instance-group-zone=asia-southeast1-b
gcloud compute backend-services add-backend be-k8s-master-k8s-api \
    --region=asia-southeast1 \
    --instance-group=k8s-master-group \
    --instance-group-zone=asia-southeast1-b
```

Create forwading rulte to create LB

```bash
gcloud compute forwarding-rules create fr-ilb-nginx \
    --region=asia-southeast1 \
    --load-balancing-scheme=internal \
    --network=k8s-net \
    --subnet=k8s-subnet \
    --address=10.240.0.6 \
    --ip-protocol=TCP \
    --ports=80 \
    --backend-service=be-k8s-master-nginx \
    --backend-service-region=asia-southeast1
gcloud compute forwarding-rules create fr-ilb-k8s-api \
    --region=asia-southeast1 \
    --load-balancing-scheme=internal \
    --network=k8s-net \
    --subnet=k8s-subnet \
    --address=10.240.0.6 \
    --ip-protocol=TCP \
    --ports=6443 \
    --backend-service=be-k8s-master-k8s-api \
    --backend-service-region=asia-southeast1
```

## References

- <https://docs.projectcalico.org/getting-started/kubernetes/self-managed-public-cloud/gce>
- <https://cloud.google.com/solutions/building-internet-connectivity-for-private-vms>
- <https://cloud.google.com/load-balancing/docs/internal/setting-up-internal>
