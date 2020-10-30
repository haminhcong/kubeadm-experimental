# Know how about how to integrate kubeadm with cloud providers

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

- <https://stackoverflow.com/questions/56354235/kubernetes-cluster-cannot-attach-and-mount-automatically-created-google-cloud-pl>

## Kubeadm with OpenStack Cloud

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

## References

- <https://github.com/kubernetes/cloud-provider-alibaba-cloud/blob/master/docs/getting-started.md>
- <https://kubernetes.io/docs/tasks/administer-cluster/running-cloud-controller/>
- <https://www.linuxschoolonline.com/how-to-set-up-kubernetes-1-16-on-aws-from-scratch/>
- <https://blog.scottlowe.org/2020/02/18/setting-up-k8s-on-aws-kubeadm-manual-certificate-distribution/>
- <https://blog.scottlowe.org/2019/08/14/setting-up-aws-integrated-kubernetes-115-cluster-kubeadm/>
