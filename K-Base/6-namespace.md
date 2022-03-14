# Kubernetes资源配置之namespace

## 简介

Kubernetes支持多个虚拟集群，这些虚拟集群依赖于同一个物理集群，这些虚拟集群就可以称作`namespace`（中文习惯称为"名称空间"或"命名空间", 个人觉得理解成"域"更为合理一些）

Namespace有资源隔离的作用，类似Linux系统的`多用户`的概念（同一物理环境,可以为多用户划分多个相互隔离的操作空间）

Namespace在多用户之间通过`资源配额（resource-quotas）`进行集群资源的划分

## 使用Namespace

```bash
# 位于namespace作用域中的资源
[root@k8s-master01 ~]# kubectl api-resources --namespaced
```

[![1][1]][1]

[1]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220313143057.png

```bash
# 不被namespace限制的资源
[root@k8s-master01 ~]# kubectl api-resources --namespaced=false
```

[![2][2]][2]

[2]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220313143135.png

### 查看Namespace

```bash
# namespace可简写为ns
[root@k8s-master01 ~]# kubectl get ns --show-kind --show-labels 
NAME                             STATUS   AGE     LABELS
namespace/default                Active   7d      kubernetes.io/metadata.name=default
namespace/kube-node-lease        Active   7d      kubernetes.io/metadata.name=kube-node-lease
namespace/kube-public            Active   7d      kubernetes.io/metadata.name=kube-public
namespace/kube-system            Active   7d      kubernetes.io/metadata.name=kube-system
```

初始状态下，Kubernetes具有四个namespace：

- default：默认的namespace，默认用户创建的资源都是在这个namespace下
- kube-node-lease：该namespace用于与各个节点相关的租约（Lease）对象；节点租期允许kubelet发送心跳，由此控制面能够检测到节点故障
- kube-public：自动创建且所有用户可读的namespace（包括未经身份认证的）
- kube-system：Kubernetes系统创建的对象所在的namespace

```bash
# 列出特定的ns
[root@k8s-master01 ~]# kubectl get ns kube-public 
NAME          STATUS   AGE
kube-public   Active   7d
# 获取特定的ns的描述信息
[root@k8s-master01 ~]# kubectl describe ns kube-public 
Name:         kube-public
Labels:       kubernetes.io/metadata.name=kube-public
Annotations:  <none>
Status:       Active

No resource quota.

No LimitRange resource.
# 如果配置了资源配额，可查看到具体信息，如下
[root@k8s-master01 ~]# kubectl describe ns quotatest 
Name:         quotatest
Labels:       kubernetes.io/metadata.name=quotatest
Annotations:  <none>
Status:       Active

Resource Quotas
  Name:     resource-test
  Resource  Used  Hard
  --------  ---   ---
  pods      1     1

Resource Limits
 Type                   Resource  Min    Max  Default Request  Default Limit  Max Limit/Request Ratio
 ----                   --------  ---    ---  ---------------  -------------  -----------------------
 Container              cpu       100m   1    500m             1              -
 Container              memory    100Mi  1Gi  256Mi            512Mi          -
 PersistentVolumeClaim  storage   1Gi    2Gi  -                -              -
# 查看namespace资源对象说明
[root@k8s-master01 ~]# kubectl explain ns
KIND:     Namespace
VERSION:  v1

DESCRIPTION:
     Namespace provides a scope for Names. Use of multiple namespaces is
     optional.

FIELDS:
   apiVersion   <string>
     APIVersion defines the versioned schema of this representation of an
     object. Servers should convert recognized schemas to the latest internal
     value, and may reject unrecognized values. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources

   kind <string>
     Kind is a string value representing the REST resource this object
     represents. Servers may infer this from the endpoint the client submits
     requests to. Cannot be updated. In CamelCase. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds

   metadata     <Object>
     Standard object metadata. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata

   spec <Object>
     Spec defines the behavior of the Namespace. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status

   status       <Object>
     Status describes the current status of a Namespace. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status
```

### 创建和删除

> 说明：避免使用前缀`kube-`创建新的namespace，因为它是为Kubernetes系统namespace保留的。

Namespace命名规则:

- 最多63个字符
- 只能包含字母数字,以及'-'
- 须以字母或数字开头
- 须以字母或数字结尾

```bash
# 1. yaml文件方式创建
[root@k8s-master01 ~]# vim new_namespace_deemoprobe.yaml 
apiVersion: v1
kind: Namespace
metadata:
  name: deemoprobe
  annotations:
    type: create by yaml
[root@k8s-master01 ~]# kubectl apply -f new_namespace_deemoprobe.yaml 
namespace/deemoprobe created
[root@k8s-master01 ~]# kubectl get ns deemoprobe --show-labels --show-kind 
NAME                   STATUS   AGE   LABELS
namespace/deemoprobe   Active   36s   kubernetes.io/metadata.name=deemoprobe

# 2. 命令创建
[root@k8s-master01 ~]# kubectl create ns deemoprobe2
namespace/deemoprobe2 created
[root@k8s-master01 ~]# kubectl get ns deemoprobe2 --show-labels --show-kind 
NAME                    STATUS   AGE   LABELS
namespace/deemoprobe2   Active   12s   kubernetes.io/metadata.name=deemoprobe2
# 删除namespace=deemoprobe2
[root@k8s-master01 ~]# kubectl delete ns deemoprobe2
namespace "deemoprobe2" deleted
[root@k8s-master01 ~]# kubectl get ns deemoprobe2 --show-labels --show-kind 
Error from server (NotFound): namespaces "deemoprobe2" not found
```

```bash
# 指定namespace操作其他在namespace作用域中的资源，如kubectl get pods --namespace=deemoprobe
# 简写 -n deemoprobe
[root@k8s-master01 ~]# kubectl get configmap -n deemoprobe 
NAME               DATA   AGE
kube-root-ca.crt   1      3m33s
```

### 切换默认namespace

```bash
# 查看当前所在namespace，新集群如果查不到，就是default
[root@k8s-master01 ~]# kubectl config view | grep namespace:
# 设置当前namespace上下文，使得相应namespace成为默认值
[root@k8s-master01 ~]# kubectl config set-context --current --namespace deemoprobe
Context "kubernetes-admin@kubernetes" modified.
[root@k8s-master01 ~]# kubectl config view | grep namespace:
    namespace: deemoprobe
# 回切
[root@k8s-master01 ~]# kubectl config set-context --current --namespace default
Context "kubernetes-admin@kubernetes" modified.
[root@k8s-master01 ~]# kubectl config view | grep namespace:
    namespace: default
```

### namespace工具

安装namespace管理工具（kubens）更快捷地管理namespace

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

比如：我们需要"测试"-"开发"-"生产"这几个环境，可以划分三个Namespace进行操作，为后续资源配额准备基础环境。

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
