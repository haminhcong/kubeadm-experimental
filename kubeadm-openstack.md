# kubeadm-gcp-experimental

Experimental to deploy k8s cluster on OpenStack Cloud Provider - VEXXHOST,
with etcdadm, kubeadm and calico

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

### Create K8S Cluster Nodes

Create 3 K8S Master Nodes and 3 K8S Worker Nodes by GUI or CLI on OpenStack
Cloud, with following name:

```log
k8s-cls-1-master-1
k8s-cls-1-master-2
k8s-cls-1-master-3

k8s-cls-1-worker-1
k8s-cls-1-worker-2
k8s-cls-1-worker-3
```

After VMs are created, update ansible inventory file with IP of Created VMs

Setup docker and prepare presequite on K8S VMs. Run this
ansible playbook command on bastion node

```bash
ansible-playbook -i inventory.ini site.yml \
    -e ansible_ssh_user=centos --key-file "/path_to_vm_key" \
    --tags "ensure_k8s_presequite"
```

SSH to each node, ensure node hostname is the same with
OpenStack Instance Name by run command:

```bash
# On node k8s-cls-1-master-1
hostnamectl set-hostname k8s-cls-1-master-1
# On node k8s-cls-1-master-2
hostnamectl set-hostname k8s-cls-1-master-2
# On node k8s-cls-1-master-3
hostnamectl set-hostname k8s-cls-1-master-3

# On node k8s-cls-1-worker-1
hostnamectl set-hostname k8s-cls-1-worker-1
# On node k8s-cls-1-worker-2
hostnamectl set-hostname k8s-cls-1-worker-2
# On node k8s-cls-1-worker-3
hostnamectl set-hostname k8s-cls-1-worker-3

```

This setup is ensure openstack cloud controller manager can recognize k8s node
via OpenStack API.

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

Create config file `kubeadm-config.yml` on k8s-controller-1 with sample from
`kubeadm-config-example.yml` file

Init k8s master:

```bash
kubeadm init --config kubeadm-config.yml
```

```setup kubectl conffig
mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Create cloud-config file from sample file `cloud.conf`
Create `cloud-config` secret in kubernetes secret:

```bash
kubectl create secret -n kube-system generic cloud-config --from-file=cloud.conf
```

Create RBAC resources and openstack-cloud-controller-manager deamonset

```bash
kubectl apply -f cloud-controller-manager-roles.yaml
kubectl apply -f cloud-controller-manager-role-bindings.yaml
kubectl apply -f openstack-cloud-controller-manager-ds.yaml
```

Create `calico.yml` manifest file for calico CNI

Apply calico CNI:

```sh
kubectl apply -f calico.yml
```

Copy k8s master certificates to `k8s-controller-2` and `k8s-controller-3`:

```bash
scp  -i /path_to_ssh_key_file /etc/kubernetes/pki/ca.crt centos@K8S_MASTER_2_IP:/home/centos
scp  -i /path_to_ssh_key_file /etc/kubernetes/pki/ca.key centos@K8S_MASTER_2_IP:/home/centos
scp  -i /path_to_ssh_key_file /etc/kubernetes/pki/sa.key centos@K8S_MASTER_2_IP:/home/centos
scp  -i /path_to_ssh_key_file /etc/kubernetes/pki/sa.pub centos@K8S_MASTER_2_IP:/home/centos
scp  -i /path_to_ssh_key_file /etc/kubernetes/pki/front-proxy-ca.crt centos@K8S_MASTER_2_IP:/home/centos
scp  -i /path_to_ssh_key_file /etc/kubernetes/pki/front-proxy-ca.key centos@K8S_MASTER_2_IP:/home/centos

scp  -i /path_to_ssh_key_file /etc/kubernetes/pki/ca.crt centos@K8S_MASTER_3_IP:/home/centos
scp  -i /path_to_ssh_key_file /etc/kubernetes/pki/ca.key centos@K8S_MASTER_3_IP:/home/centos
scp  -i /path_to_ssh_key_file /etc/kubernetes/pki/sa.key centos@K8S_MASTER_3_IP:/home/centos
scp  -i /path_to_ssh_key_file /etc/kubernetes/pki/sa.pub centos@K8S_MASTER_3_IP:/home/centos
scp  -i /path_to_ssh_key_file /etc/kubernetes/pki/front-proxy-ca.crt centos@K8S_MASTER_3_IP:/home/centos
scp  -i /path_to_ssh_key_file /etc/kubernetes/pki/front-proxy-ca.key centos@K8S_MASTER_3_IP:/home/centos
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

Create join config file with following sample format

```yaml
apiVersion: kubeadm.k8s.io/v1beta2
discovery:
  bootstrapToken:
    apiServerEndpoint: 10.240.230.50:6443
    token: JOIN_TOKEN
    caCertHashes: ["CA_CERT_HASH"]
kind: JoinConfiguration
nodeRegistration:
  kubeletExtraArgs:
    cloud-provider: "external"
controlPlane:
  localAPIEndpoint:
    advertiseAddress: K8S_MASTER_2_OR_3_IP_ADDRESS
```

Run join command

```bash
kubeadm join --config kubeadm-config.yml
```

### Setup K8S Workers

Perform join k8s worker in each k8s-worker node by run this command with root user:

```log
apiVersion: kubeadm.k8s.io/v1beta2
discovery:
  bootstrapToken:
    apiServerEndpoint: 10.240.230.50:6443
    token: JOIN_TOKEN
    caCertHashes: ["CA_CERT_HASH"]
kind: JoinConfiguration
nodeRegistration:
  kubeletExtraArgs:
    cloud-provider: "external"
```

Run join command

```bash
kubeadm join --config kubeadm-config.yml
```

## Test Cluster After Setup Done

Create a k8s nginx pod:

```yaml
# nginx.yaml file content

apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
   app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.17.3-alpine
    ports:
    - containerPort: 80
```

```bash
kubectl apply -f nginx.yaml
```

Create a nginx LoadBalancer Service:

```yaml
# nginx-service.yaml file content
apiVersion: v1
kind: Service
metadata:
  name: nginxservice
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
  selector:
    app: nginx
  type: LoadBalancer
```

```bash
kubectl apply -f nginx-service.yaml
```

Verify K8S Service is created

```bash
kubectl get services
NAME           TYPE           CLUSTER-IP        EXTERNAL-IP    PORT(S)        AGE
kubernetes     ClusterIP      192.168.128.1     <none>         443/TCP        11h
nginxservice   LoadBalancer   192.168.213.173   38.108.68.29   80:32279/TCP   10m
```

Try access service:

```bash
curl -L http://38.108.68.29:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
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
