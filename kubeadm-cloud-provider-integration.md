# Know how about how to integrate kubeadm with cloud providers

## Kube Controller Manager Flags

- https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/
- https://godoc.org/k8s.io/kubernetes/cmd/kubeadm/app/apis/kubeadm/v1beta1

## Kubeadm with Alibaba Cloud

`kubeadm.conf` for Alibaba Cloud: <https://github.com/kubernetes/cloud-provider-alibaba-cloud/blob/master/docs/examples/kubeadm-new.conf>

```yaml
apiVersion: kubeadm.k8s.io/v1alpha3
kind: InitConfiguration
bootstrapTokens:
- token: zlrecy.1umn2p8qhjuh5bwk
nodeRegistration:
  kubeletExtraArgs:
    cloud-provider: external
  name: cn-hongkong.i-j6c0zd30oi7a5el8au1m
---
apiVersion: kubeadm.k8s.io/v1alpha3
kind: ClusterConfiguration
imageRepository: registry-vpc.${region}.aliyuncs.com/acs
apiServerExtraArgs:
  cloud-provider: external
clusterName: kubernetes
controllerManagerExtraArgs:
  cloud-provider: external
  horizontal-pod-autoscaler-use-rest-clients: "false"
  node-cidr-mask-size: "20"
networking:
  dnsDomain: cluster.local
  podSubnet: 172.16.0.0/16
  serviceSubnet: 172.19.0.0/20
apiServerExtraVolumes:
- hostPath: /etc/localtime
  mountPath: /etc/localtime
  name: localtime
controllerManagerExtraVolumes:
- hostPath: /etc/localtime
  mountPath: /etc/localtime
  name: localtime
schedulerExtraVolumes:
- hostPath: /etc/localtime
  mountPath: /etc/localtime
  name: localtime
```

## Kubeadm with Google Cloud

Ref:

- <https://medium.com/dev-genius/how-to-kubectl-loadbalancer-and-configure-nginx-ingress-controller-for-a-gcp-cluster-bc2f2599f8f1>
- <https://docs.openshift.com/container-platform/3.10/install_config/configuring_gce.html#gce-configuring-masters-manual_configuring-for-GCE>
- <https://serverfault.com/questions/943450/gcp-load-balancer-not-created-when-deployment-is-exposed>

```conf
$ sudo vi /etc/kubernetes/gce.conf
[Global]
project-id = rosy-cache-200605
node-tags = kubernetes
key = AIzaSy*****************************p62GkA
```

```yaml
---
apiVersion: kubeadm.k8s.io/v1beta1
kind: InitConfiguration
bootstrapTokens:
- token:
  ttl: "0"
nodeRegistration:
 kubeletExtraArgs:
  cloud-provider: "external"
  cloud-config: /etc/kubernetes/gce.conf
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.18.0
apiServer:
  extraArgs:
    enable-admission-plugins: NodeRestriction,AlwaysPullImages,DefaultStorageClass
    authorization-mode: Node,RBAC
controllerManager:
  extraArgs:
    allocate-node-cidrs: "true"
    cluster-cidr: 10.244.0.0/16
    cloud-config: /etc/kubernetes/gce.conf
    enable-taint-manager: "false"
    cloud-provider: "external"
```

- <https://stackoverflow.com/questions/56354235/kubernetes-cluster-cannot-attach-and-mount-automatically-created-google-cloud-pl>

```conf
[Global]
project-id = "my-beautiful-project-39276"
node-tags = nodeports
node-instance-prefix = "kubernetes-node"
multizone = true
```

```yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: InitConfiguration
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: medium.howtok5678songce
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
nodeRegistration:
  criSocket: "/var/run/containerd/containerd.sock"
  kubeletExtraArgs:
    cloud-provider: "gce"
    cloud-config: "/etc/kubernetes/cloud-config"
  taints: []
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
clusterName: medium
kubernetesVersion: v1.16.2
networking:
  podSubnet: 10.244.0.0/16
apiServer:
  certSANs:
  - 35.187.224.267
  extraArgs:
    authorization-mode: Node,RBAC
    cloud-provider: "gce"
    cloud-config: "/etc/kubernetes/cloud-config"
  extraVolumes:
  - name: cloud
    hostPath: "/etc/kubernetes/cloud-config"
    mountPath: "/etc/kubernetes/cloud-config"
controllerManager:
  extraArgs:
    cloud-provider: "gce"
    cloud-config: "/etc/kubernetes/cloud-config"
  extraVolumes:
  - name: cloud
    hostPath: "/etc/kubernetes/cloud-config"
    mountPath: "/etc/kubernetes/cloud-config"
```

## Kubeadm with OpenStack Cloud

Ref:

- <https://kubernetes.io/blog/2020/02/07/deploying-external-openstack-cloud-provider-with-kubeadm/>
- <https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/cinder-csi-plugin/examples.md>
- <https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/openstack-cloud-controller-manager/using-openstack-cloud-controller-manager.md>

```yaml
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: InitConfiguration
nodeRegistration:
  kubeletExtraArgs:
    cloud-provider: "external"

---
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.17.0
networking:
  podSubnet: 192.168.0.0/16
apiServer:
  extraArgs:
    cloud-provider: "external"
controllerManager:
  extraArgs:
    cloud-provider: "external"
```

```conf
[Global]
auth-url=http://192.128.1.15:5000/v3
#Tip: You can also use Application Credential ID and Secret in place of username, password, tenant-id, and domain-id.
#application-credential-id=
#application-credential-secret=
username=admin
# user-id=
password=passw0rd
region=RegionOne
tenant-id=
domain-id=

[LoadBalancer]
use-octavia=true
subnet-id=9c018814-f2a8-45dd-9679-5400d49b333a
floating-network-id=80c9fc78-a2be-45d5-a85b-f17a6c98df92

[BlockStorage]
bs-version=v2
```

```yaml
apiVersion: kubeadm.k8s.io/v1beta1
kind: InitConfiguration
nodeRegistration:
  kubeletExtraArgs:
    cloud-provider: "external"
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: "v1.15.1"
apiServer:
  extraArgs:
    enable-admission-plugins: NodeRestriction
    runtime-config: "storage.k8s.io/v1=true"
controllerManager:
  extraArgs:
    external-cloud-volume-plugin: openstack
  extraVolumes:
  - name: "cloud-config"
    hostPath: "/etc/kubernetes/cloud-config"
    mountPath: "/etc/kubernetes/cloud-config"
    readOnly: true
    pathType: File
networking:
  serviceSubnet: "10.96.0.0/12"
  podSubnet: "10.224.0.0/16"
  dnsDomain: "cluster.local"
```

## Kubeadm with Amazon AWS Cloud

References:

- <https://blog.scottlowe.org/2020/02/18/setting-up-k8s-on-aws-kubeadm-manual-certificate-distribution/>
- <https://blog.scottlowe.org/2019/08/14/setting-up-aws-integrated-kubernetes-115-cluster-kubeadm/>

`kubeadm.conf` for AWS

```yaml
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
apiServer:
  extraArgs:
    cloud-provider: aws
clusterName:
controlPlaneEndpoint: cp-nlb.elb.us-east-2.amazonaws.com
controllerManager:
  extraArgs:
    cloud-provider: aws
    configure-cloud-routes: "false"
kubernetesVersion: v1.17.0
networking:
  podSubnet: 192.168.0.0/16
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: InitConfiguration
nodeRegistration:
  kubeletExtraArgs:
    cloud-provider: aws
```

```yaml
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: JoinConfiguration
discovery:
  bootstrapToken:
    token: 123456.i2v5ub39hrvf51j3
    apiServerEndpoint: cp-nlb.elb.us-east-2.amazonaws.com
    caCertHashes:
      - "sha256:082feed98fb5fd2b497472fb7d9553414e27ff7eeb7b919c82ff3a08fdf5782f"
nodeRegistration:
  name: ip-10-10-100-155.us-east-2.compute.internal
  kubeletExtraArgs:
    cloud-provider: aws
controlPlane:
  localAPIEndpoint:
    advertiseAddress: 10.10.100.155
```

## Kubeadm with vsphere

Ref: <https://blah.cloud/kubernetes/setting-up-k8s-and-the-vsphere-cloud-provider-using-kubeadm/>

```yaml
apiVersion: kubeadm.k8s.io/v1beta1
kind: InitConfiguration
bootstrapTokens:
       - groups:
         - system:bootstrappers:kubeadm:default-node-token
         token: y7yaev.9dvwxx6ny4ef8vlq
         ttl: 0s
         usages:
         - signing
         - authentication
nodeRegistration:
  kubeletExtraArgs:
    cloud-provider: "vsphere"
    cloud-config: "/etc/kubernetes/vsphere.conf"
---
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
kubernetesVersion: v1.13.3
apiServer:
  extraArgs:
    cloud-provider: "vsphere"
    cloud-config: "/etc/kubernetes/vsphere.conf"
  extraVolumes:
  - name: cloud
    hostPath: "/etc/kubernetes/vsphere.conf"
    mountPath: "/etc/kubernetes/vsphere.conf"
controllerManager:
  extraArgs:
    cloud-provider: "vsphere"
    cloud-config: "/etc/kubernetes/vsphere.conf"
  extraVolumes:
  - name: cloud
    hostPath: "/etc/kubernetes/vsphere.conf"
    mountPath: "/etc/kubernetes/vsphere.conf"
networking:
  podSubnet: "10.244.0.0/16"
```

## Kubeadm with Azure

Ref: https://blog.jreypo.io/containers/microsoft/azure/cloud/cloud-native/devops/deploying-a-kubernetes-cluster-in-azure-using-kubeadm/

```yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: InitConfiguration
nodeRegistration:
  kubeletExtraArgs:
    cloud-provider: "azure"
    cloud-config: "/etc/kubernetes/cloud.conf"
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.13.0
apiServer:
  extraArgs:
    cloud-provider: "azure"
    cloud-config: "/etc/kubernetes/cloud.conf"
  extraVolumes:
  - name: cloud
    hostPath: "/etc/kubernetes/cloud.conf"
    mountPath: "/etc/kubernetes/cloud.conf"
controllerManager:
  extraArgs:
    cloud-provider: "azure"
    cloud-config: "/etc/kubernetes/cloud.conf"
  extraVolumes:
  - name: cloud
    hostPath: "/etc/kubernetes/cloud.conf"
    mountPath: "/etc/kubernetes/cloud.conf"
networking:
  serviceSubnet: "10.12.0.0/16"
  podSubnet: "10.11.0.0/16"
```

## References

- <https://github.com/kubernetes/cloud-provider-alibaba-cloud/blob/master/docs/getting-started.md>
- <https://kubernetes.io/docs/tasks/administer-cluster/running-cloud-controller/>
- <https://www.linuxschoolonline.com/how-to-set-up-kubernetes-1-16-on-aws-from-scratch/>
- <https://blog.scottlowe.org/2020/02/18/setting-up-k8s-on-aws-kubeadm-manual-certificate-distribution/>
- <https://blog.scottlowe.org/2019/08/14/setting-up-aws-integrated-kubernetes-115-cluster-kubeadm/>
