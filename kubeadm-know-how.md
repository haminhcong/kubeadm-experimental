# Some knowhow about kubeadm

**WARNING: Use with caution change some config will break your cluster**

## Reconfigure kubeadm kubelet in live cluster

Source: https://github.com/kubernetes/kubeadm/issues/1464#issuecomment-518021866

Ref: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/kubelet-integration/

- using kubectl edit modify the kubelet-config-x.yy ConfigMap in the cluster.
- on all nodes open the file /var/lib/kubelet/config.yaml make your modifications and restart the kubelet using sudo systemctl restart kubelet

## Reconfigure kubeadm k8s controller components in live cluster

### Method 1

Source: <https://github.com/kubernetes/kubeadm/issues/1464#issuecomment-518021984>

1. edit the kubeadm-config ConfigMap using kubectl edit
2. go to all nodes and modify /etc/kubernetes/manifests/*.yaml

### Method 2

Sources:

- <https://blog.honosoft.com/2020/01/31/kubeadm-how-to-upgrade-update-your-configuration/>
- <https://stackoverflow.com/questions/49810966/how-to-use-kubeadm-upgrade-to-change-some-features-in-kubeadm-config>

1. Steps 1: Update Configuration File
1. Steps 2: Redeploy on kubeadm by kubeadm upgrade command:

```shell
kubeadm upgrade apply --config /etc/kubeadm.yaml
```

## Customizing control plane configuration with kubeadm

Use `ClusterConfiguration` object. Example

```yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.16.0
apiServer:
  extraArgs:
    advertise-address: 192.168.0.103
    anonymous-auth: "false"
    enable-admission-plugins: AlwaysPullImages,DefaultStorageClass
    audit-log-path: /home/johndoe/audit.log
controllerManager:
  extraArgs:
    cluster-signing-key-file: /home/johndoe/keys/ca.key
    bind-address: 0.0.0.0
    deployment-controller-sync-period: "50"
scheduler:
  extraArgs:
    address: 0.0.0.0
    config: /home/johndoe/schedconfig.yaml
    kubeconfig: /home/johndoe/kubeconfig.yaml
```


Sources: <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/control-plane-flags/>

## Kubeadm phases internal design

- https://kubernetes.io/docs/reference/setup-tools/kubeadm/implementation-details/
- https://github.com/kubernetes/kubeadm/blob/master/docs/design/design_v1.10.md

## Kubeadm Source Code Analysis

- https://github.com/kubernetes/kubernetes/tree/master/cmd/kubeadm
- https://programming.vip/docs/kubeadm-source-code-analysis.html