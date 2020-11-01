# Some knowledge about Kubernetes Cloud Controller Managers

## What is (External) Cloud Controller Manager

Cloud Controller Manager is introduced to separate functionality of interact with cloud provider from core of Kubernetes
to another component developed by third-party cloud provider outside core of Kubernetes. Example:

- <https://github.com/kubernetes/cloud-provider-alibaba-cloud>
- <https://github.com/kubernetes/cloud-provider-openstack>
- <https://github.com/digitalocean/digitalocean-cloud-controller-manager/blob/master/releases/v0.1.4.yml>

Each cloud provider has them own Cloud Controller Provider act as interface between Kubernetes and this Cloud Provider.

## Development

As offical document of Kubernetes, the specific Cloud Controller Manager for a Cloud Provider need to be
developed in a separate repository outside k8s core repository. Development can started with support from `cloud-provider` package built by kuberntes community: <https://github.com/kubernetes/cloud-provider>. This package provides number of Interfaces which a Cloud Controller Manager 
need to implement due to it's functionality, with main Interface is: [Interface interface](https://github.com/kubernetes/cloud-provider/blob/release-1.17/cloud.go#L43)

## Deployment

After development done, cloud controller manager is built to a indepent container image. Examples:

- `docker.io/k8scloudprovider/openstack-cloud-controller-manager:latest`
- `digitalocean/digitalocean-cloud-controller-manager:v0.1.30`

Then Cloud Controller Manager is deployed as a DaemonSet/Pod running a control lop for cloud-specific control logic in containers. When deploy K8S Operator need to set RBAC for cloud controller manager,
as well as provide Cloud Configurations when deploy Cloud Controller Manager DaemonSet. Example:

```yaml
# Alibaba Cloud Controller Manager DaemonSet
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  labels:
    app: cloud-controller-manager
    tier: control-plane
  name: cloud-controller-manager
  namespace: kube-system
spec:
    # ...
    spec:
    # ...
    nodeSelector:
         node-role.kubernetes.io/master: ""
      containers:
      - command:
        -  /cloud-controller-manager
        - --kubeconfig=/etc/kubernetes/cloud-controller-manager.conf
        - --address=127.0.0.1
        - --allow-untagged-cloud=true
        - --leader-elect=true
        - --cloud-provider=alicloud
        - --allocate-node-cidrs=true
        - --cluster-cidr=${CLUSTER_CIDR}
        - --use-service-account-credentials=true
        - --route-reconciliation-period=30s
        - --v=5
        image: registry.cn-hangzhou.aliyuncs.com/acs/cloud-controller-manager-amd64:v1.9.3.10-gfb99107-aliyun
        env:
        - name: ACCESS_KEY_ID
          valueFrom:
            configMapKeyRef:
              name: cloud-config
              key: special.keyid
        - name: ACCESS_KEY_SECRET
          valueFrom:
            configMapKeyRef:
              name: cloud-config
              key: special.keysecret
```

```yaml
---
# OpenStack Cloud Controller Manager DaemonSet
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: openstack-cloud-controller-manager
  namespace: kube-system
  labels:
    k8s-app: openstack-cloud-controller-manager
spec:
  selector:
    matchLabels:
      k8s-app: openstack-cloud-controller-manager
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        k8s-app: openstack-cloud-controller-manager
    spec:
    # ...
      serviceAccountName: cloud-controller-manager
      containers:
        - name: openstack-cloud-controller-manager
          image: docker.io/k8scloudprovider/openstack-cloud-controller-manager:latest
          args:
            - /bin/openstack-cloud-controller-manager
            - --v=1
            - --cloud-config=$(CLOUD_CONFIG)
            - --cloud-provider=openstack
            - --use-service-account-credentials=true
            - --bind-address=127.0.0.1
          volumeMounts:
            - mountPath: /etc/kubernetes/pki
              name: k8s-certs
              readOnly: true
            - mountPath: /etc/ssl/certs
              name: ca-certs
              readOnly: true
            - mountPath: /etc/config
              name: cloud-config-volume
              readOnly: true
          resources:
            requests:
              cpu: 200m
          env:
            - name: CLOUD_CONFIG
              value: /etc/config/cloud.conf
```

```conf
# OpenStack Cloud Controller Manager configuration files
# /etc/config/cloud.conf
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

To adopt with external cloud controller manager, `kube-api-server` and `kube-controller-manager` should
not run any cloud specific loop control logic. Kubelet must run with `--cloud-provider=external`

## Some notes about in-tree cloud manager controller

We could run in-tree cloud controller manager as external cloud controller manager:

<https://kubernetes.io/docs/tasks/administer-cluster/running-cloud-controller/#examples>

[In-tree cloud provider depracation plan](https://github.com/kubernetes/enhancements/blob/master/keps/sig-cloud-provider/20190125-removing-in-tree-providers.md):

- Move all code in `k8s.io/kubernetes/pkg/cloudprovider/providers/<provider>` to `k8s.io/kubernetes/staging/src/k8s.io/legacy-cloud-providers/<provider>/`. This requires removing all internal dependencies in each cloud provider to k8s.io/kubernetes.
- Begin to build/release the CCM from external repos (`k8s.io/cloud-provider-<provider>`) with the option to import the legacy providers from `k8s.io/legacy-cloud-providers/<provider>`. This allows the cloud-controller-manager to opt into legacy behavior in-tree (for compatibility reasons) or build new implementations of the provider. Development for cloud providers in-tree is still done in `k8s.io/kubernetes/staging/src/k8s.io/legacy-cloud-providers/<provider>` during this phase.
- Delete all code in `k8s.io/kubernetes/staging/src/k8s.io/legacy-cloud-providers` and shift main development to `k8s.io/cloud-provider-<provider>`. External cloud provider repos can optionally still import `k8s.io/legacy-cloud-providers` but it will no longer be imported from core components in k8s.io/kubernetes.

In current time of this notes (01/11/2020) this plan is in phase 2

## References

- <https://medium.com/@m.json/the-kubernetes-cloud-controller-manager-d440af0d2be5>
- <https://kubernetes.io/docs/concepts/architecture/cloud-controller/>
- <https://kubernetes.io/docs/tasks/administer-cluster/running-cloud-controller/>
- <https://kubernetes.io/docs/tasks/administer-cluster/developing-cloud-controller-manager/>
- <https://unofficial-kubernetes.readthedocs.io/en/latest/tasks/administer-cluster/running-cloud-controller/>
- <https://github.com/kubernetes/enhancements/blob/master/keps/sig-cloud-provider/20190125-removing-in-tree-providers.md>
- <https://github.com/kubernetes/community/tree/master/sig-cloud-provider>
- <https://github.com/kubernetes/enhancements/blob/master/keps/sig-cloud-provider/20180530-cloud-controller-manager.md>
