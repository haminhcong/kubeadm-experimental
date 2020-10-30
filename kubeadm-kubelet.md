# Some Knowledge About Kubeadm

## Kubeadm Kubernetes Master Configuration files

 - `/etc/kubernetes/manifests` as the path where kubelet should look for static Pod manifests
   - Temporarily when bootstrapping, these manifests are present:
     - `etcd.yaml`
     - `kube-apiserver.yaml`
     - `kube-controller-manager.yaml`
     - `kube-scheduler.yaml`
 - `/etc/kubernetes/kubelet.conf` as the path where the kubelet should store its credentials to the API server.
 - `/etc/kubernetes/admin.conf` as the path from where the admin can fetch his/her superuser credentials.
 - Names of certificates files:
   - `ca.crt`, `ca.key` (CA certificate)
   - `apiserver.crt`, `apiserver.key` (API server certificate)
   - `apiserver-kubelet-client.crt`, `apiserver-kubelet-client.key` (client certificate for the apiservers to connect to the kubelets securely)
   - `sa.pub`, `sa.key` (a private key for signing ServiceAccount )
   - `front-proxy-ca.crt`, `front-proxy-ca.key` (CA for the front proxy)
   - `front-proxy-client.crt`, `front-proxy-client.key` (client cert for the front proxy client)
- Names of kubeconfig files:
  - `admin.conf`
  - `kubelet.conf` (`bootstrap-kubelet.conf` during TLS bootstrap)
  - `controller-manager.conf`
  - `scheduler.conf`

## Kubeadm Init Process

<https://github.com/kubernetes/kubeadm/blob/master/docs/design/design_v1.10.md#kubeadm-init-phases>

## Kubeadm Kubelet Important Config Files

Source: <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/kubelet-integration/>

- /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf or `/etc/systemd/system/kubelet.service.d/10-kubeadm.conf`:
  Main config file, created when run package install command. eg `yum -y install kubelet`

This file specifies the default locations for all of the files managed by kubeadm for the kubelet.

- The KubeConfig file to use for the TLS Bootstrap is `/etc/kubernetes/bootstrap-kubelet.conf`,
  but it is only used if `/etc/kubernetes/kubelet.conf` does not exist.
- The KubeConfig file with the unique kubelet identity is `/etc/kubernetes/kubelet.conf`.
- The file containing the kubelet's ComponentConfig is `/var/lib/kubelet/config.yaml`.
- The dynamic environment file that contains `KUBELET_KUBEADM_ARGS` is sourced from `/var/lib/kubelet/kubeadm-flags.env`.
- The file that can contain user-specified flag overrides with `KUBELET_EXTRA_ARGS` is sourced from
  `/etc/default/kubelet` (for DEBs), or `/etc/sysconfig/kubelet` (for RPMs). `KUBELET_EXTRA_ARGS`
  is last in the flag chain and has the highest priority in the event of conflicting settings.

Moreover, kubelet config is also saved in Kubernetes Config Map `kubelet-config-1.X`,
where X is the minor version of the Kubernetes version you are initializing. Example with kubernetes 1.19.3

```bash
kubectl describe configmap/kubelet-config-1.19 -n kube-system
Name:         kubelet-config-1.19
Namespace:    kube-system
Labels:       <none>
Annotations:  kubeadm.kubernetes.io/component-config.hash: sha256:8381625d7b7c8ec3f7b5f3f27668a0ca6df9c9dd7de75e87925758629d14c79c

Data
====
kubelet:
----
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 0s
    cacheUnauthorizedTTL: 0s
cgroupDriver: systemd
clusterDNS:
- 192.168.128.10
clusterDomain: cluster.local
cpuManagerReconcilePeriod: 0s
evictionPressureTransitionPeriod: 0s
fileCheckFrequency: 0s
healthzBindAddress: 127.0.0.1
healthzPort: 10248
httpCheckFrequency: 0s
imageMinimumGCAge: 0s
kind: KubeletConfiguration
logging: {}
nodeStatusReportFrequency: 0s
nodeStatusUpdateFrequency: 0s
rotateCertificates: true
runtimeRequestTimeout: 0s
staticPodPath: /etc/kubernetes/manifests
streamingConnectionIdleTimeout: 0s
syncFrequency: 0s
volumeStatsAggPeriod: 0s
Events:  <none>

```

```log
systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /usr/lib/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: active (running) since T3 2020-10-27 14:01:06 UTC; 3 days ago
     Docs: https://kubernetes.io/docs/
 Main PID: 6525 (kubelet)
    Tasks: 14
   Memory: 111.7M
   CGroup: /system.slice/kubelet.service
           └─6525 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconf...

```

Kubeadm kubelet config file generate process

With `kubeadm init`

```log
When you call kubeadm init, the kubelet configuration is marshalled to disk at /var/lib/kubelet/config.yaml, and also uploaded to a ConfigMap in the cluster. The ConfigMap is named kubelet-config-1.X, where X is the minor version of the Kubernetes version you are initializing. A kubelet configuration file is also written to /etc/kubernetes/kubelet.conf with the baseline cluster-wide configuration for all kubelets in the cluster. This configuration file points to the client certificates that allow the kubelet to communicate with the API server. This addresses the need to propagate cluster-level configuration to each kubelet.
```

## Kubelet Dynamic Configuration

- <https://kubernetes.io/docs/tasks/administer-cluster/reconfigure-kubelet/>

## References

- [https://github.com/kubernetes/kubeadm/blob/master/docs/design/design_v1.10.md#a-note-on-constants--well-known-values-and-paths](https://github.com/kubernetes/kubeadm/blob/master/docs/design/design_v1.10.md#a-note-on-constants--well-known-values-and-paths)
- <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/kubelet-integration/>