# Kubernetes资源配置之namespace

## 何为namesapce?

Kubernetes支持多个虚拟集群, 这些虚拟集群依赖于同一个物理集群, 这些虚拟集群就可以称作`namespace`(中文习惯称为"名称空间"或"命名空间", 其实理解成"域"更为合理一些).

Namespace有资源隔离的作用, 类似Linux系统的`多用户`的概念(同一物理环境,可以为多用户划分多个相互隔离的操作空间,互不影响).

Namespace在多用户之间通过`资源配额(resource-quotas)`进行集群资源的划分.

## 使用Namespace

首先指明一点: 大部分API资源都位于特定的namespace中, 但并不是所有的API资源都在特定的namespace中(如: namespace本身/持久卷PV等)

```shell
# 位于namespace中的资源
[root@k8s-master ~]# kubectl api-resources --namespaced=true
NAME                        SHORTNAMES   APIGROUP                    NAMESPACED   KIND
bindings                                                             true         Binding
configmaps                  cm                                       true         ConfigMap
endpoints                   ep                                       true         Endpoints
events                      ev                                       true         Event
limitranges                 limits                                   true         LimitRange
persistentvolumeclaims      pvc                                      true         PersistentVolumeClaim
pods                        po                                       true         Pod
podtemplates                                                         true         PodTemplate
replicationcontrollers      rc                                       true         ReplicationController
resourcequotas              quota                                    true         ResourceQuota
secrets                                                              true         Secret
serviceaccounts             sa                                       true         ServiceAccount
services                    svc                                      true         Service
controllerrevisions                      apps                        true         ControllerRevision
daemonsets                  ds           apps                        true         DaemonSet
deployments                 deploy       apps                        true         Deployment
replicasets                 rs           apps                        true         ReplicaSet
statefulsets                sts          apps                        true         StatefulSet
localsubjectaccessreviews                authorization.k8s.io        true         LocalSubjectAccessReview
horizontalpodautoscalers    hpa          autoscaling                 true         HorizontalPodAutoscaler
cronjobs                    cj           batch                       true         CronJob
jobs                                     batch                       true         Job
leases                                   coordination.k8s.io         true         Lease
networkpolicies                          crd.projectcalico.org       true         NetworkPolicy
networksets                              crd.projectcalico.org       true         NetworkSet
endpointslices                           discovery.k8s.io            true         EndpointSlice
events                      ev           events.k8s.io               true         Event
ingresses                   ing          extensions                  true         Ingress
ingresses                   ing          networking.k8s.io           true         Ingress
networkpolicies             netpol       networking.k8s.io           true         NetworkPolicy
poddisruptionbudgets        pdb          policy                      true         PodDisruptionBudget
rolebindings                             rbac.authorization.k8s.io   true         RoleBinding
roles                                    rbac.authorization.k8s.io   true         Role
# 不在namespace中的资源
[root@k8s-master ~]# kubectl api-resources --namespaced=false
NAME                              SHORTNAMES   APIGROUP                       NAMESPACED   KIND
componentstatuses                 cs                                          false        ComponentStatus
namespaces                        ns                                          false        Namespace
nodes                             no                                          false        Node
persistentvolumes                 pv                                          false        PersistentVolume
mutatingwebhookconfigurations                  admissionregistration.k8s.io   false        MutatingWebhookConfiguration
validatingwebhookconfigurations                admissionregistration.k8s.io   false        ValidatingWebhookConfiguration
customresourcedefinitions         crd,crds     apiextensions.k8s.io           false        CustomResourceDefinition
apiservices                                    apiregistration.k8s.io         false        APIService
tokenreviews                                   authentication.k8s.io          false        TokenReview
selfsubjectaccessreviews                       authorization.k8s.io           false        SelfSubjectAccessReview
selfsubjectrulesreviews                        authorization.k8s.io           false        SelfSubjectRulesReview
subjectaccessreviews                           authorization.k8s.io           false        SubjectAccessReview
certificatesigningrequests        csr          certificates.k8s.io            false        CertificateSigningRequest
bgpconfigurations                              crd.projectcalico.org          false        BGPConfiguration
bgppeers                                       crd.projectcalico.org          false        BGPPeer
blockaffinities                                crd.projectcalico.org          false        BlockAffinity
clusterinformations                            crd.projectcalico.org          false        ClusterInformation
felixconfigurations                            crd.projectcalico.org          false        FelixConfiguration
globalnetworkpolicies                          crd.projectcalico.org          false        GlobalNetworkPolicy
globalnetworksets                              crd.projectcalico.org          false        GlobalNetworkSet
hostendpoints                                  crd.projectcalico.org          false        HostEndpoint
ipamblocks                                     crd.projectcalico.org          false        IPAMBlock
ipamconfigs                                    crd.projectcalico.org          false        IPAMConfig
ipamhandles                                    crd.projectcalico.org          false        IPAMHandle
ippools                                        crd.projectcalico.org          false        IPPool
kubecontrollersconfigurations                  crd.projectcalico.org          false        KubeControllersConfiguration
ingressclasses                                 networking.k8s.io              false        IngressClass
runtimeclasses                                 node.k8s.io                    false        RuntimeClass
podsecuritypolicies               psp          policy                         false        PodSecurityPolicy
clusterrolebindings                            rbac.authorization.k8s.io      false        ClusterRoleBinding
clusterroles                                   rbac.authorization.k8s.io      false        ClusterRole
priorityclasses                   pc           scheduling.k8s.io              false        PriorityClass
csidrivers                                     storage.k8s.io                 false        CSIDriver
csinodes                                       storage.k8s.io                 false        CSINode
storageclasses                    sc           storage.k8s.io                 false        StorageClass
volumeattachments                              storage.k8s.io                 false        VolumeAttachment
```

### 查看Namespace

```shell
# namespace可简写为ns
[root@k8s-master ~]# kubectl get ns
NAME              STATUS   AGE
default           Active   55d
kube-node-lease   Active   55d
kube-public       Active   55d
kube-system       Active   55d
test              Active   11d
```

初始状态下，Kubernetes 具有三个名字空间：

- default 默认的namespace,默认情况下用户创建的资源都是在这个namespace下.
- kube-system 由 Kubernetes 系统创建的对象的namespace.
- kube-public 自动创建且被所有用户可读的namespace(包括未经身份认证的).此namespace通常某些资源在整个集群可公开读取被集群使用.
- kube-node-lease 此名字空间用于与各个节点相关的租期(Lease)对象; 此对象的设计使得集群规模很大时节点心跳检测性能得到提升.

```shell
# 列出特定的ns
[root@k8s-master ~]# kubectl get ns test
NAME   STATUS   AGE
test   Active   11d
# 获取特定的ns的描述信息
# 如果配置了资源配额和资源限制, 可以看到
[root@k8s-master ~]# kubectl describe ns test
Name:         test
Labels:       <none>
Annotations:  <none>
Status:       Active

No resource quota.

No LimitRange resource.
```

### 创建和删除

> 说明： 避免使用前缀 kube- 创建namespace，因为它是为 Kubernetes 系统namespace保留的.

Namespace命名规则:

- 最多63个字符
- 只能包含字母数字，以及'-'
- 须以字母或数字开头
- 须以字母或数字结尾

```shell
# 1. yaml文件方式创建
[root@k8s-master apiserver]# vi namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: deemoprobe
[root@k8s-master apiserver]# kubectl apply -f namespace.yaml 
namespace/deemoprobe created
[root@k8s-master apiserver]# kubectl get ns
NAME              STATUS   AGE
deemoprobe        Active   6s
default           Active   55d
kube-node-lease   Active   55d
kube-public       Active   55d
kube-system       Active   55d
test              Active   11d
# 2. 命令创建
[root@k8s-master apiserver]# kubectl create ns deemoprobe01
namespace/deemoprobe01 created
[root@k8s-master apiserver]# kubectl get ns
NAME              STATUS   AGE
deemoprobe        Active   86s
deemoprobe01      Active   2s
default           Active   55d
kube-node-lease   Active   55d
kube-public       Active   55d
kube-system       Active   55d
test              Active   11d
# 删除namespace=test
[root@k8s-master ~]# kubectl delete ns test
namespace "test" deleted
[root@k8s-master ~]# kubectl get ns
NAME              STATUS   AGE
deemoprobe        Active   160m
deemoprobe01      Active   159m
default           Active   55d
kube-node-lease   Active   55d
kube-public       Active   55d
kube-system       Active   55d
```

### namespace

```shell
# 为请求指定namespace, 例如kubectl get pods --namespace=deemoprobe
[root@k8s-master ~]# kubectl get po -n=deemoprobe
No resources found in deemoprobe namespace.
# 查看当前所在namespace
[root@k8s-master ~]# kubectl config view | grep namespace:
    namespace: default
# 切换namespace
[root@k8s-master ~]# kubectl config set-context --current --namespace=deemoprobe
Context "kubernetes-admin@kubernetes" modified.
[root@k8s-master ~]# kubectl config view | grep namespace:
    namespace: deemoprobe
```

安装namespace切换工具, 实现

```shell
[root@k8s-master ~]# curl -L https://github.com/ahmetb/kubectx/releases/download/v0.9.1/kubens -o /bin/kubens
[root@k8s-master ~]# chmod +x /bin/kubens
[root@k8s-master /]# kubens -h
USAGE:
  kubens                    : list the namespaces in the current context
  kubens <NAME>             : change the active namespace of current context
  kubens -                  : switch to the previous namespace in this context
  kubens -c, --current      : show the current namespace
  kubens -h,--help          : show this message
# 列出所有namespace
[root@k8s-master ~]# kubens
deemoprobe
deemoprobe01
default
kube-node-lease
kube-public
kube-system
# 查看当前namespace
[root@k8s-master ~]# kubens -c
default
# 切换为deemoprobe
[root@k8s-master ~]# kubens deemoprobe
Context "kubernetes-admin@kubernetes" modified.
Active namespace is "deemoprobe".
[root@k8s-master ~]# kubens -c
deemoprobe
# 切换为上一个namespace
[root@k8s-master ~]# kubens -
Context "kubernetes-admin@kubernetes" modified.
Active namespace is "default".
[root@k8s-master ~]# kubens -c
default
```

## 使用场景

比如: 我们需要"测试"-"开发"-"生产"这几个环境, 可以划分三个Namespace进行操作, 为资源配额准备环境.

```shell
[root@k8s-master apiserver]# vi namespace.yaml 
---
apiVersion: v1
kind: Namespace
metadata:
  name: test
  labels:
    name: test
---
apiVersion: v1
kind: Namespace
metadata:
  name: dev
  labels:
    name: dev
---
apiVersion: v1
kind: Namespace
metadata:
  name: prod
  labels:
    name: prod
[root@k8s-master apiserver]# kubectl apply -f namespace.yaml 
namespace/test created
namespace/dev created
namespace/prod created
[root@k8s-master apiserver]# kubectl get ns --show-labels
NAME              STATUS   AGE   LABELS
default           Active   55d   <none>
dev               Active   64s   name=dev
kube-node-lease   Active   55d   <none>
kube-public       Active   55d   <none>
kube-system       Active   55d   <none>
prod              Active   64s   name=prod
test              Active   65s   name=test
# 全新的Namespace, 未设置任何资源配额和限制
[root@k8s-master apiserver]# kubectl describe ns test
Name:         test
Labels:       name=test
Annotations:  <none>
Status:       Active

No resource quota.

No LimitRange resource.
[root@k8s-master apiserver]# kubectl describe ns dev
Name:         dev
Labels:       name=dev
Annotations:  <none>
Status:       Active

No resource quota.

No LimitRange resource.
[root@k8s-master apiserver]# kubectl describe ns prod
Name:         prod
Labels:       name=prod
Annotations:  <none>
Status:       Active

No resource quota.

No LimitRange resource.
```
