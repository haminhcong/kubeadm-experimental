# kubeadm-gcp-experimental

Experimental to deploy k8s cluster on VEXXHOST with etcdadm, kubeadm and calico

## Presequite

- Add Account SSH Key for Project Wide Scope with user `centos`

## Cluster Component

![cluster-component](./images/cluster-component.png)

## IP Address Planning

- Cluster Network: VPC `10.240.230.0/24`
- K8S serviceSubnet: 192.168.128.0/17
- K8S podSubnet: 192.168.0.0/17

## Steps to setup

### Create Resources

Create Private VPC Network: "10.240.230.0/24"

Create bastion host, add network tag and allow ssh from ssh to bastion host.

### Setup bastion host

- Install ansible
- Install git
- Clone this repo to bastion host

### Setup firewall rules

Allow SSH Connection:

```bash
openstack security group rule create default \
      --ingress --protocol tcp --dst-port 22:22 --remote-ip 0.0.0.0/0

```

Allow connection between hosts in cluster, and from hosts in cluster to external network/internet

```bash

openstack security group rule create default \
      --ingress --protocol tcp --dst-port 1:65535 --remote-group default
openstack security group rule create default \
      --ingress --protocol udp --dst-port 1:65535 --remote-group default
openstack security group rule create default \
      --egress --protocol tcp --dst-port 1:65535 --remote-ip 0.0.0.0/0
openstack security group rule create default \
      --egress --protocol udp --dst-port 1:65535 --remote-ip 0.0.0.0/0

```

Allow connection from K8S Load Balancer to host etcd endpoint and host k8s-api endpoint

```bash
openstack security group rule create default \
      --ingress --protocol tcp --dst-port 2379:2379 --remote-ip 10.240.230.0/24
openstack security group rule create default \
      --ingress --protocol tcp --dst-port 6443:6443 --remote-ip 10.240.230.0/24
```

Allow IP-IP Protocol of Calico CNI:

```bash
openstack  security group rule create  --protocol 4  --egress  default
openstack  security group rule create  --protocol 4  --ingress  default
```

### Setup K8S API LB

Create LB

```bash
openstack loadbalancer create --name k8s-api-lb --vip-address 10.240.230.50

```

Create Listener

```bash
openstack loadbalancer listener create --protocol TCP --protocol-port 2379 --name k8s-etcd-listener  <LB_ID>
openstack loadbalancer listener create --protocol TCP --protocol-port 6443 --name k8s-api-listener  <LB_ID>

```

Create LB Pool and Pool Health Monitor

```bash
openstack loadbalancer pool create --name k8s-etcd-pool --listener <k8s-etcd-listener-id> --protocol TCP --lb-algorithm ROUND_ROBIN
openstack loadbalancer pool create --name k8s-api-pool --listener <k8s-api-listener-id> --protocol TCP --lb-algorithm ROUND_ROBIN
openstack loadbalancer healthmonitor create --delay 5 --max-retries 4 --timeout 10 --type TCP k8s-etcd-pool
openstack loadbalancer healthmonitor create --delay 5 --max-retries 4 --timeout 10 --type TCP k8s-api-pool

```

Create Pool Member

```bash
openstack loadbalancer member create \
    --name k8s-cls1-master-1 --weight 1 --address 10.240.230.K8S-MASTER-1-IP \
    --subnet-id <k8s_host_subnet_id> --protocol-port 2379 \
    k8s-etcd-pool

openstack loadbalancer member create \
    --name k8s-cls1-master-2 --weight 1 --address 10.240.230.K8S-MASTER-2-IP \
    --subnet-id <k8s_host_subnet_id> --protocol-port 2379 \
    k8s-etcd-pool

openstack loadbalancer member create \
    --name k8s-cls1-master-3 --weight 1 --address 10.240.230.K8S-MASTER-3-IP \
    --subnet-id <k8s_host_subnet_id> --protocol-port 2379 \
    k8s-etcd-pool


openstack loadbalancer member create \
    --name k8s-cls1-master-1 --weight 1 --address 10.240.230.K8S-MASTER-1-IP \
    --subnet-id <k8s_host_subnet_id> --protocol-port 6443 \
    k8s-api-pool

openstack loadbalancer member create \
    --name k8s-cls1-master-2 --weight 1 --address 10.240.230.K8S-MASTER-2-IP \
    --subnet-id <k8s_host_subnet_id> --protocol-port 6443 \
    k8s-api-pool

openstack loadbalancer member create \
    --name k8s-cls1-master-3 --weight 1 --address 10.240.230.K8S-MASTER-3-IP \
    --subnet-id <k8s_host_subnet_id> --protocol-port 6443 \
    k8s-api-pool
```

### Setup K8S Masters

Create three K8S Master VMs: `10.240.0.11, 10.240.0.12, 10.240.0.13`

```bash
gcloud compute instances create k8s-controller-1 \
    --async \
    --boot-disk-size 40GB \
    --boot-disk-type pd-ssd \
    --can-ip-forward \
    --image-family centos-7 \
    --image-project centos-cloud \
    --machine-type e2-medium \
    --no-address \
    --private-network-ip 10.240.0.11 \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet k8s-subnet \
    --zone asia-southeast1-b \
    --tags k8s-cluster,controller,etcd

gcloud compute instances create k8s-controller-2 \
    --async \
    --boot-disk-size 40GB \
    --boot-disk-type pd-ssd \
    --can-ip-forward \
    --image-family centos-7 \
    --image-project centos-cloud \
    --machine-type e2-medium \
    --private-network-ip 10.240.0.12 \
    --no-address \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet k8s-subnet \
    --zone asia-southeast1-b \
    --tags k8s-cluster,controller,etcd

gcloud compute instances create k8s-controller-3 \
    --async \
    --boot-disk-size 40GB \
    --boot-disk-type pd-ssd \
    --can-ip-forward \
    --image-family centos-7 \
    --image-project centos-cloud \
    --machine-type e2-medium \
    --private-network-ip 10.240.0.13 \
    --no-address \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet k8s-subnet \
    --zone asia-southeast1-b \
    --tags k8s-cluster,controller,etcd
```

Setup docker and prepare presequite on 3 K8S Master VMs

```bash
ansible-playbook -i inventory.ini site.yml -e ansible_ssh_user=centos --key-file "/PATH_TO_GOOGLE_CLOUD_VM_KEY" --tags "ensure_k8s_presequite"
```

#### Setup etcd cluster

Init etcd cluster in `k8s-controller-1` by run following commands

```bash
# Run following command with root user
wget https://github.com/kubernetes-sigs/etcdadm/releases/download/v0.1.3/etcdadm-linux-amd64
mv etcdadm-linux-amd64 /usr/local/sbin/etcdadm
chmod +x  /usr/local/sbin/etcdadm
ETCDCTL_API=3 etcdadm init --version 3.4.13
```

Copy etcd certs from `k8s-controller-1` to `k8s-controller-2` and `k8s-controller-3`:

```bash
scp -i /PATH_TO_SSH_KEY /etc/etcd/pki/ca.* centos@10.240.0.12:/home/centos
scp -i /PATH_TO_SSH_KEY /etc/etcd/pki/ca.* centos@10.240.0.13:/home/centos
```

Join `k8s-controller-2` and `k8s-controller-3` to etcd cluster by run following commands
with root user on them:

```bash

mkdir -p /etc/etcd/pki/
mv /home/centos/ca* /etc/etcd/pki/

wget https://github.com/kubernetes-sigs/etcdadm/releases/download/v0.1.3/etcdadm-linux-amd64
mv etcdadm-linux-amd64 /usr/local/sbin/etcdadm
chmod +x  /usr/local/sbin/etcdadm

ETCDCTL_API=3 etcdadm join --version 3.4.13 https://10.240.0.11:2379
```

#### Setup k8s-controller-1 k8s controller components

use root user, create`kubeadm-config.yml` file with token is generated from command `kubeadm token generate`

Create config file `kubeadm-config.yml` on k8s-controller-1

Init k8s master:

```bash
kubeadm init --config kubeadm-config.yml
```

```setup kubectl conffig
mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Create `calico.yml` manifest file for calico CNI

Apply calico CNI:

```sh
kubectl apply -f calico.yml
```

Copy k8s master certificates to `k8s-controller-2` and `k8s-controller-3`:

```bash
scp  -i /PATH_TO_GOOGLE_CLOUD_VM_KEY /etc/kubernetes/pki/ca.crt centos@10.240.0.12:/home/centos
scp  -i /PATH_TO_GOOGLE_CLOUD_VM_KEY /etc/kubernetes/pki/ca.key centos@10.240.0.12:/home/centos
scp  -i /PATH_TO_GOOGLE_CLOUD_VM_KEY /etc/kubernetes/pki/sa.key centos@10.240.0.12:/home/centos
scp  -i /PATH_TO_GOOGLE_CLOUD_VM_KEY /etc/kubernetes/pki/sa.pub centos@10.240.0.12:/home/centos
scp  -i /PATH_TO_GOOGLE_CLOUD_VM_KEY /etc/kubernetes/pki/front-proxy-ca.crt centos@10.240.0.12:/home/centos
scp  -i /PATH_TO_GOOGLE_CLOUD_VM_KEY /etc/kubernetes/pki/front-proxy-ca.key centos@10.240.0.12:/home/centos

scp  -i /PATH_TO_GOOGLE_CLOUD_VM_KEY /etc/kubernetes/pki/ca.crt centos@10.240.0.13:/home/centos
scp  -i /PATH_TO_GOOGLE_CLOUD_VM_KEY /etc/kubernetes/pki/ca.key centos@10.240.0.13:/home/centos
scp  -i /PATH_TO_GOOGLE_CLOUD_VM_KEY /etc/kubernetes/pki/sa.key centos@10.240.0.13:/home/centos
scp  -i /PATH_TO_GOOGLE_CLOUD_VM_KEY /etc/kubernetes/pki/sa.pub centos@10.240.0.13:/home/centos
scp  -i /PATH_TO_GOOGLE_CLOUD_VM_KEY /etc/kubernetes/pki/front-proxy-ca.crt centos@10.240.0.13:/home/centos
scp  -i /PATH_TO_GOOGLE_CLOUD_VM_KEY /etc/kubernetes/pki/front-proxy-ca.key centos@10.240.0.13:/home/centos

```

#### Setup k8s-controller-2 and k8s-controller-3

```bash

USER=centos # customizable
mkdir -p /etc/kubernetes/pki/etcd
mv /home/${USER}/ca.crt /etc/kubernetes/pki/
mv /home/${USER}/ca.key /etc/kubernetes/pki/
mv /home/${USER}/sa.pub /etc/kubernetes/pki/
mv /home/${USER}/sa.key /etc/kubernetes/pki/
mv /home/${USER}/front-proxy-ca.crt /etc/kubernetes/pki/
mv /home/${USER}/front-proxy-ca.key /etc/kubernetes/pki/
```

Perform join k8s control plane:

```log
kubeadm join 10.240.0.6:6443 --token KUBE_CLUSTER_TOKEN --discovery-token-ca-cert-hash sha256:KUBE_CLUSTER_CA_CERT_HASH --control-plane
```

After join successful, add k8s-master-2 and k8s-master-3 to master
instance group:

```bash
gcloud compute instance-groups unmanaged add-instances k8s-master-group \
    --zone=asia-southeast1-b \
    --instances=k8s-controller-2,k8s-controller-3
```

### Setup K8S Workers

Create three K8S Master VMs: `10.240.0.21, 10.240.0.22, 10.240.0.23`

```bash
gcloud compute instances create k8s-worker-1 \
    --async \
    --boot-disk-size 40GB \
    --can-ip-forward \
    --image-family centos-7 \
    --image-project centos-cloud \
    --machine-type e2-medium \
    --no-address \
    --private-network-ip 10.240.0.21 \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet k8s-subnet \
    --zone asia-southeast1-b \
    --tags k8s-cluster,worker

gcloud compute instances create k8s-worker-2 \
    --async \
    --boot-disk-size 40GB \
    --can-ip-forward \
    --image-family centos-7 \
    --image-project centos-cloud \
    --machine-type e2-medium \
    --private-network-ip 10.240.0.22 \
    --no-address \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet k8s-subnet \
    --zone asia-southeast1-b \
    --tags k8s-cluster,worker

gcloud compute instances create k8s-worker-3 \
    --async \
    --boot-disk-size 40GB \
    --can-ip-forward \
    --image-family centos-7 \
    --image-project centos-cloud \
    --machine-type e2-medium \
    --private-network-ip 10.240.0.23 \
    --no-address \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet k8s-subnet \
    --zone asia-southeast1-b \
    --tags k8s-cluster,worker
```

Setup docker and prepare presequite on 3 K8S Master VMs

```bash
ansible-playbook -i inventory.ini site.yml -e ansible_ssh_user=centos --key-file "/PATH_TO_GOOGLE_CLOUD_VM_KEY" --tags "ensure_k8s_presequite"
```

Perform join k8s worker in each k8s-worker node by run this command with root user:

```log
kubeadm join 10.240.0.6:6443 --token KUBE_CLUSTER_TOKEN --discovery-token-ca-cert-hash sha256:KUBE_CLUSTER_CA_CERT_HASH
```

## References

- <https://cloud.google.com/compute/docs/instances/adding-removing-ssh-keys#project-wide>
- <https://docs.projectcalico.org/getting-started/kubernetes/self-managed-public-cloud/gce>
- <https://cloud.google.com/solutions/building-internet-connectivity-for-private-vms>
- <https://cloud.google.com/load-balancing/docs/internal/setting-up-internal>
- <https://github.com/kubernetes-sigs/etcdadm>
- <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/>
- <https://medium.com/faun/kubernetes-spin-up-highly-available-kubernetes-cluster-using-kubeadm-setup-cni-part-3-6af4f53aa735>
- <https://unofficial-kubernetes.readthedocs.io/en/latest/admin/kubeadm/>
- <https://www.devops.buzz/public/kubeadm/change-servicesubnet-cidr>
