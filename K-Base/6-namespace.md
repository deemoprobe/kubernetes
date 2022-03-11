# Kubernetes资源配置之namespace

## 简介

Kubernetes支持多个虚拟集群, 这些虚拟集群依赖于同一个物理集群, 这些虚拟集群就可以称作`namespace`(中文习惯称为"名称空间"或"命名空间", 个人觉得理解成"域"更为合理一些).

Namespace有资源隔离的作用, 类似Linux系统的`多用户`的概念(同一物理环境,可以为多用户划分多个相互隔离的操作空间).

Namespace在多用户之间通过`资源配额(resource-quotas)`进行集群资源的划分.

## 使用Namespace

```bash
# 位于namespace作用域中的资源
[root@k8s-master01 ~]# kubectl api-resources --namespaced=true
NAME                        SHORTNAMES   APIVERSION                     NAMESPACED   KIND
bindings                                 v1                             true         Binding
configmaps                  cm           v1                             true         ConfigMap
endpoints                   ep           v1                             true         Endpoints
events                      ev           v1                             true         Event
limitranges                 limits       v1                             true         LimitRange
persistentvolumeclaims      pvc          v1                             true         PersistentVolumeClaim
pods                        po           v1                             true         Pod
podtemplates                             v1                             true         PodTemplate
replicationcontrollers      rc           v1                             true         ReplicationController
resourcequotas              quota        v1                             true         ResourceQuota
secrets                                  v1                             true         Secret
serviceaccounts             sa           v1                             true         ServiceAccount
services                    svc          v1                             true         Service
controllerrevisions                      apps/v1                        true         ControllerRevision
daemonsets                  ds           apps/v1                        true         DaemonSet
deployments                 deploy       apps/v1                        true         Deployment
replicasets                 rs           apps/v1                        true         ReplicaSet
statefulsets                sts          apps/v1                        true         StatefulSet
localsubjectaccessreviews                authorization.k8s.io/v1        true         LocalSubjectAccessReview
horizontalpodautoscalers    hpa          autoscaling/v2                 true         HorizontalPodAutoscaler
cronjobs                    cj           batch/v1                       true         CronJob
jobs                                     batch/v1                       true         Job
leases                                   coordination.k8s.io/v1         true         Lease
networkpolicies                          crd.projectcalico.org/v1       true         NetworkPolicy
networksets                              crd.projectcalico.org/v1       true         NetworkSet
endpointslices                           discovery.k8s.io/v1            true         EndpointSlice
events                      ev           events.k8s.io/v1               true         Event
pods                                     metrics.k8s.io/v1beta1         true         PodMetrics
ingresses                   ing          networking.k8s.io/v1           true         Ingress
networkpolicies             netpol       networking.k8s.io/v1           true         NetworkPolicy
poddisruptionbudgets        pdb          policy/v1                      true         PodDisruptionBudget
rolebindings                             rbac.authorization.k8s.io/v1   true         RoleBinding
roles                                    rbac.authorization.k8s.io/v1   true         Role
csistoragecapacities                     storage.k8s.io/v1beta1         true         CSIStorageCapacity
# 不被namespace限制的资源
[root@k8s-master01 ~]# kubectl api-resources --namespaced=false
NAME                              SHORTNAMES   APIVERSION                             NAMESPACED   KIND
componentstatuses                 cs           v1                                     false        ComponentStatus
namespaces                        ns           v1                                     false        Namespace
nodes                             no           v1                                     false        Node
persistentvolumes                 pv           v1                                     false        PersistentVolume
mutatingwebhookconfigurations                  admissionregistration.k8s.io/v1        false        MutatingWebhookConfiguration
validatingwebhookconfigurations                admissionregistration.k8s.io/v1        false        ValidatingWebhookConfiguration
customresourcedefinitions         crd,crds     apiextensions.k8s.io/v1                false        CustomResourceDefinition
apiservices                                    apiregistration.k8s.io/v1              false        APIService
tokenreviews                                   authentication.k8s.io/v1               false        TokenReview
selfsubjectaccessreviews                       authorization.k8s.io/v1                false        SelfSubjectAccessReview
selfsubjectrulesreviews                        authorization.k8s.io/v1                false        SelfSubjectRulesReview
subjectaccessreviews                           authorization.k8s.io/v1                false        SubjectAccessReview
certificatesigningrequests        csr          certificates.k8s.io/v1                 false        CertificateSigningRequest
bgpconfigurations                              crd.projectcalico.org/v1               false        BGPConfiguration
bgppeers                                       crd.projectcalico.org/v1               false        BGPPeer
blockaffinities                                crd.projectcalico.org/v1               false        BlockAffinity
clusterinformations                            crd.projectcalico.org/v1               false        ClusterInformation
felixconfigurations                            crd.projectcalico.org/v1               false        FelixConfiguration
globalnetworkpolicies                          crd.projectcalico.org/v1               false        GlobalNetworkPolicy
globalnetworksets                              crd.projectcalico.org/v1               false        GlobalNetworkSet
hostendpoints                                  crd.projectcalico.org/v1               false        HostEndpoint
ipamblocks                                     crd.projectcalico.org/v1               false        IPAMBlock
ipamconfigs                                    crd.projectcalico.org/v1               false        IPAMConfig
ipamhandles                                    crd.projectcalico.org/v1               false        IPAMHandle
ippools                                        crd.projectcalico.org/v1               false        IPPool
kubecontrollersconfigurations                  crd.projectcalico.org/v1               false        KubeControllersConfiguration
flowschemas                                    flowcontrol.apiserver.k8s.io/v1beta2   false        FlowSchema
prioritylevelconfigurations                    flowcontrol.apiserver.k8s.io/v1beta2   false        PriorityLevelConfiguration
nodes                                          metrics.k8s.io/v1beta1                 false        NodeMetrics
ingressclasses                                 networking.k8s.io/v1                   false        IngressClass
runtimeclasses                                 node.k8s.io/v1                         false        RuntimeClass
podsecuritypolicies               psp          policy/v1beta1                         false        PodSecurityPolicy
clusterrolebindings                            rbac.authorization.k8s.io/v1           false        ClusterRoleBinding
clusterroles                                   rbac.authorization.k8s.io/v1           false        ClusterRole
priorityclasses                   pc           scheduling.k8s.io/v1                   false        PriorityClass
csidrivers                                     storage.k8s.io/v1                      false        CSIDriver
csinodes                                       storage.k8s.io/v1                      false        CSINode
storageclasses                    sc           storage.k8s.io/v1                      false        StorageClass
volumeattachments                              storage.k8s.io/v1                      false        VolumeAttachment
```

### 查看Namespace

```bash
# namespace可简写为ns
[root@k8s-master01 ~]# kubectl get ns
NAME                   STATUS   AGE
default                Active   28d
kube-node-lease        Active   28d
kube-public            Active   28d
kube-system            Active   28d
```

初始状态下,Kubernetes 具有三个名字空间：

- default 默认的namespace,默认情况下用户创建的资源都是在这个namespace下.
- kube-node-lease 此名字空间用于与各个节点相关的租期(Lease)对象; 此对象的设计使得集群规模很大时节点心跳检测性能得到提升.
- kube-public 自动创建且被所有用户可读的namespace(包括未经身份认证的).此namespace通常某些资源在整个集群可公开读取被集群使用.
- kube-system 由 Kubernetes 系统创建的对象的namespace.

```bash
# 列出特定的ns
[root@k8s-master01 ~]# kubectl get ns default
NAME      STATUS   AGE
default   Active   28d
# 获取特定的ns的描述信息
# 如果配置了资源配额和资源限制, 可以看到
[root@k8s-master01 ~]# kubectl describe ns default
Name:         default
Labels:       kubernetes.io/metadata.name=default
Annotations:  <none>
Status:       Active

No resource quota.

No LimitRange resource.
```

### 创建和删除

> 说明： 避免使用前缀 kube- 创建namespace,因为它是为 Kubernetes 系统namespace保留的.

Namespace命名规则:

- 最多63个字符
- 只能包含字母数字,以及'-'
- 须以字母或数字开头
- 须以字母或数字结尾

```bash
# 1. yaml文件方式创建
[root@k8s-master01 ~]# vim namespace_deemoprobe.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: deemoprobe
[root@k8s-master01 ~]# kubectl create -f namespace_deemoprobe.yaml 
namespace/deemoprobe created
[root@k8s-master01 ~]# kubectl get ns
NAME                   STATUS   AGE
deemoprobe             Active   4s
default                Active   28d
kube-node-lease        Active   28d
kube-public            Active   28d
kube-system            Active   28d
# 2. 命令创建
[root@k8s-master01 ~]# kubectl create ns test
namespace/test created
[root@k8s-master01 ~]# kubectl get ns
NAME                   STATUS   AGE
deemoprobe             Active   50s
default                Active   28d
kube-node-lease        Active   28d
kube-public            Active   28d
kube-system            Active   28d
test                   Active   4s
# 删除namespace=test
[root@k8s-master01 ~]# kubectl delete ns test
namespace "test" deleted
```

```bash
# 为请求指定namespace, 例如kubectl get pods --namespace=deemoprobe
[root@k8s-master01 ~]# kubectl get cm -n deemoprobe
NAME               DATA   AGE
kube-root-ca.crt   1      2m23s
# 查看当前所在namespace，新集群如果查不到，就是default
[root@k8s-master01 ~]# kubectl config view | grep namespace:
    namespace: default
# 切换namespace
[root@k8s-master01 ~]# kubectl config set-context --current --namespace=deemoprobe
Context "kubernetes-admin@kubernetes" modified.
[root@k8s-master01 ~]# kubectl config view | grep namespace:
    namespace: deemoprobe
```

### namespace工具

安装namespace切换工具（kubens）, 实现

```bash
[root@k8s-master01 ~]# curl -L https://github.com/ahmetb/kubectx/releases/download/v0.9.4/kubens -o /bin/kubens
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   643  100   643    0     0    163      0  0:00:03  0:00:03 --:--:--   163
100  5555  100  5555    0     0    638      0  0:00:08  0:00:08 --:--:--  1704
[root@k8s-master01 ~]# chmod +x /bin/kubens 
[root@k8s-master01 ~]# kubens -h
USAGE:
  kubens                    : list the namespaces in the current context
  kubens <NAME>             : change the active namespace of current context
  kubens -                  : switch to the previous namespace in this context
  kubens -c, --current      : show the current namespace
  kubens -h,--help          : show this message
# 列出所有namespace，
[root@k8s-master01 ~]# kubens
deemoprobe
default
kube-node-lease
kube-public
kube-system
# 查看当前namespace
[root@k8s-master01 ~]# kubens -c
default
# 切换为deemoprobe
[root@k8s-master01 ~]# kubens deemoprobe
Context "kubernetes-admin@kubernetes" modified.
Active namespace is "deemoprobe".
[root@k8s-master01 ~]# kubens -c
deemoprobe
# 切换为上一个namespace
[root@k8s-master01 ~]# kubens -
Context "kubernetes-admin@kubernetes" modified.
Active namespace is "default".
[root@k8s-master01 ~]# kubens -c
default
```

## 使用场景

比如: 我们需要"测试"-"开发"-"生产"这几个环境, 可以划分三个Namespace进行操作, 为资源配额准备环境.

```bash
[root@k8s-master01 ~]# vi namespace.yaml 
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
[root@k8s-master01 ~]# kubectl apply -f namespace.yaml 
namespace/test created
namespace/dev created
namespace/prod created
[root@k8s-master01 ~]# kubectl get ns --show-labels
NAME              STATUS   AGE   LABELS
dev               Active   64s   name=dev
prod              Active   64s   name=prod
test              Active   65s   name=test
...
# 全新的Namespace, 未设置任何资源配额和限制
[root@k8s-master01 ~]# kubectl describe ns test
Name:         test
Labels:       name=test
Annotations:  <none>
Status:       Active

No resource quota.

No LimitRange resource.
[root@k8s-master01 ~]# kubectl describe ns dev
Name:         dev
Labels:       name=dev
Annotations:  <none>
Status:       Active

No resource quota.

No LimitRange resource.
[root@k8s-master01 ~]# kubectl describe ns prod
Name:         prod
Labels:       name=prod
Annotations:  <none>
Status:       Active

No resource quota.

No LimitRange resource.
```
