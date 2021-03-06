# Kubernetes策略之资源配额

场景: 当多用户共享同一集群节点时, 为了避免集群中相关资源的不合理使用和提高资源的利用率, 对该集群的资源进行合理分配和限制, Kubernetes中通过"资源配额"解决这种问题

通过 `ResourceQuota` 对象定义,对每个Namespace的资源消耗总量提供限制. 可以限制Namespace中某种类型的对象的总数目上限,也可以限制Namespace中资源对象对计算机资源的使用量(CPU/内存/磁盘空间等).

ResourceQuota 对象命名规则:

- 不能超过253个字符
- 只能包含字母数字,以及'-' 和 '.'
- 须以字母或数字开头
- 须以字母或数字结尾

在集群容量小于各命名空间配额总和的情况下,可能存在资源竞争.资源竞争时,Kubernetes 系统会遵循先到先得的原则.

不管是资源竞争还是配额的修改,都不会影响已经创建的资源使用对象.

## 1. 启用资源配额

ResourceQuota默认是开启的, 对用户来说是无感的(在配置文件里不显示). 如果没有启用, 可在默认的apiserver配置文件`/etc/kubernetes/manifests/kube-apiserver.yaml`文件中`command`下面的`kube-apiserver --enable-admission-plugins`参数后添加`ResourceQuota`(多个配置间逗号间隔)

文件编辑后apiserver的Pod会自动重启, 重启后相关组件会自动启用.

```shell
[root@k8s-master manifests]# cat kube-apiserver.yaml 
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubeadm.kubernetes.io/kube-apiserver.advertise-address.endpoint: 192.168.43.10:6443
  creationTimestamp: null
  labels:
    component: kube-apiserver
    tier: control-plane
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=192.168.43.10
    - --allow-privileged=true
    - --authorization-mode=Node,RBAC
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --enable-admission-plugins=NodeRestriction,ResourceQuota
...
# 可见编辑后已自动重启运行
[root@k8s-master manifests]# kubectl get po -n kube-system
NAME                                       READY   STATUS    RESTARTS   AGE
...
kube-apiserver-k8s-master                  1/1     Running   0          2m47s
...
# 查看apiserver描述信息
[root@k8s-master manifests]# kubectl describe po kube-apiserver-k8s-master  
Name:                 kube-apiserver-k8s-master
Namespace:            kube-system
Priority:             2000001000
Priority Class Name:  system-node-critical
Node:                 k8s-master/192.168.43.10
Start Time:           Tue, 05 Jan 2021 13:41:21 +0800
Labels:               component=kube-apiserver
                      tier=control-plane
Annotations:          kubeadm.kubernetes.io/kube-apiserver.advertise-address.endpoint: 192.168.43.10:6443
                      kubernetes.io/config.hash: a021f897aa2ffbeab3a8e83605f479e3
                      kubernetes.io/config.mirror: a021f897aa2ffbeab3a8e83605f479e3
                      kubernetes.io/config.seen: 2021-01-06T15:05:29.935026940+08:00
                      kubernetes.io/config.source: file
Status:               Running
IP:                   192.168.43.10
IPs:
  IP:           192.168.43.10
Controlled By:  Node/k8s-master
Containers:
  kube-apiserver:
...
    Command:
      kube-apiserver
...
      --enable-admission-plugins=NodeRestriction,ResourceQuota
...
```

## 2. CPU和Memory资源配额

> 当然也可以对GPU进行配额(如: `requests.nvidia.com/gpu: 4`)

支持的资源类型:

| 资源名称        | 描述信息                                                         |
| --------------- | ---------------------------------------------------------------- |
| limits.cpu      | 所有非终止状态的 Pod,其 CPU 限额总量不能超过该值.                |
| limits.memory   | 所有非终止状态的 Pod,其内存限额总量不能超过该值.                 |
| requests.cpu    | 所有非终止状态的 Pod,其 CPU 需求总量不能超过该值.                |
| requests.memory | 所有非终止状态的 Pod,其内存需求总量不能超过该值.                 |
| hugepages-size  | 对于所有非终止状态的 Pod,针对指定尺寸的巨页请求总数不能超过此值. |
| cpu             | 与 requests.cpu 相同.                                            |
| memory          | 与 requests.memory 相同.                                         |

> CPU单位换算：100m CPU,100 milliCPU 和 0.1 CPU 都相同；精度不能超过 1m.1000m CPU = 1 CPU.

## 3. 存储资源配额

可以对特定的Namespace下的存储资源总量进行限制.也可以根据相关的存储类(Storage Class)来限制存储资源

| 资源名称                                                              | 描述                                                                    |
| --------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| requests.storage                                                      | 所有 PVC,存储资源的需求总量不能超过该值.                                |
| persistentvolumeclaims                                                | 在该命名空间中所允许的 PVC 总量.                                        |
| storage-class-name.storageclass.storage.k8s.io/requests.storage       | 在所有与 storage-class-name 相关的所有PVC中,存储请求的总和不能超过该值. |
| storage-class-name.storageclass.storage.k8s.io/persistentvolumeclaims | 在与 storage-class-name 相关的所有PVC中,命名空间中可以存在的PVC总数.    |

## 4. 资源对象数目配额

集群中存在过多的 Secret 实际上会导致服务器和控制器无法启动. 用户可以选择对 Job 进行配额管理,以防止配置不当的 CronJob 在某命名空间中创建太多 Job 而导致集群拒绝服务.或者考虑到有限的资源空间限制命名空间下Pod的数量等.

支持以下类型：

| 资源名称               | 描述                                                                                                                   |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| configmaps             | 在该命名空间中允许存在的 ConfigMap 总数上限.                                                                           |
| persistentvolumeclaims | 在该命名空间中允许存在的 PVC 的总数上限.                                                                               |
| pods                   | 在该命名空间中允许存在的非终止状态的 Pod 总数上限.Pod 终止状态等价于 Pod 的 .status.phase in (Failed, Succeeded) 为真. |
| replicationcontrollers | 在该命名空间中允许存在的 ReplicationController 总数上限.                                                               |
| resourcequotas         | 在该命名空间中允许存在的 ResourceQuota 总数上限.                                                                       |
| services               | 在该命名空间中允许存在的 Service 总数上限.                                                                             |
| services.loadbalancers | 在该命名空间中允许存在的 LoadBalancer 类型的 Service 总数上限.                                                         |
| services.nodeports     | 在该命名空间中允许存在的 NodePort 类型的 Service 总数上限.                                                             |
| secrets                | 在该命名空间中允许存在的 Secret 总数上限.                                                                              |

使用以下语法对所有标准的、命名空间域的资源类型进行配额设置：

- `count/<resource>.<group>`：用于非核心(core)组的资源
- `count/<resource>`：用于核心组的资源

## 5. 配额作用域

每个配额都有一组相关的 scope(作用域),配额只会对作用域内的资源生效. 配额机制仅统计所列举的作用域的交集中的资源用量.

当一个作用域被添加到配额中后,它会对作用域相关的资源数量作限制. 如配额中指定了允许(作用域)集合之外的资源,会导致验证错误.

| 作用域         | 描述                                                 |
| -------------- | ---------------------------------------------------- |
| Terminating    | 匹配所有 spec.activeDeadlineSeconds 不小于 0 的 Pod. |
| NotTerminating | 匹配所有 spec.activeDeadlineSeconds 是 nil 的 Pod.   |
| BestEffort     | 匹配所有 QoS 是 BestEffort 的 Pod.                   |
| NotBestEffort  | 匹配所有 QoS 不是 BestEffort 的 Pod.                 |
| PriorityClass  | 匹配所有引用了所指定的优先级类的 Pods.               |

BestEffort 跟踪以下资源：

- pods

Terminating、NotTerminating、NotBestEffort 和 PriorityClass 跟踪以下资源：

- pods
- cpu
- memory
- requests.cpu
- requests.memory
- limits.cpu
- limits.memory

scopeSelector字段进行作用域的配置.

定义 scopeSelector 时,如果使用以下值之一作为 scopeName 的值,则对应的 operator 只能是 Exists.

- Terminating
- NotTerminating
- BestEffort
- NotBestEffort

scopeSelector 支持 PriorityClass 在 operator 字段中使用以下值：

- In
- NotIn
- Exists
- DoesNotExist

如果 operator 是 In 或 NotIn 之一,则 values 字段必须至少包含一个值.operator 为 Exists 或 DoesNotExist,则不可以设置 values 字段. 例如：

```shell
scopeSelector:
matchExpressions:
    - scopeName: PriorityClass
    operator: In
    values:
        - high
---
scopeSelector:
matchExpressions:
    - scopeName: Terminating
    operator: Exists
```

## 6. 资源配额实例

## 7. 基于优先级的调度

Pod 可以创建为特定的优先级. 通过使用资源配额中的 scopeSelector 字段,用户可以根据 Pod 的优先级控制其系统资源消耗.

如果配额对象通过 scopeSelector 字段设置其作用域为优先级类,则配额对象只能跟踪以下资源：

- pods
- cpu
- memory
- ephemeral-storage
- limits.cpu
- limits.memory
- limits.ephemeral-storage
- requests.cpu
- requests.memory
- requests.ephemeral-storage

实例说明:

- 创建三个优先级的资源配额: "low"、"medium"、"high"
- 匹配特定的Pod

```shell
# 获取apiVersion, resourcesquota简写为quota
[root@k8s-master apiserver]# kubectl explain quota
KIND:     ResourceQuota
VERSION:  v1

DESCRIPTION:
     ResourceQuota sets aggregate quota restrictions enforced per namespace

FIELDS:
...
# 创建三个优先级资源配额
[root@k8s-master apiserver]# vi quota.yml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pods-high
spec:
  hard:
    cpu: "100"
    memory: 10Gi
    pods: "10"
  scopeSelector:
    matchExpressions:
    - scopeName: PriorityClass
      operator: In
      values:
      - high
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pods-medium
spec:
  hard:
    cpu: "50"
    memory: 5Gi
    pods: "10"
  scopeSelector:
    matchExpressions:
    - scopeName: PriorityClass
      operator: In
      values:          
      - medium
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pods-low
spec:
  hard:
    cpu: "10"
    memory: 1Gi
    pods: "10"
  scopeSelector:
    matchExpressions:
    - scopeName: PriorityClass
      operator: In
      values:
      - low
# 创建
[root@k8s-master apiserver]# kubectl apply -f quota.yml 
resourcequota/pods-high created
resourcequota/pods-medium created
resourcequota/pods-low created
# 查看
[root@k8s-master apiserver]# kubectl get quota
NAME          AGE   REQUEST                                  LIMIT
pods-high     34s   cpu: 0/100, memory: 0/10Gi, pods: 0/10   
pods-low      34s   cpu: 0/10, memory: 0/1Gi, pods: 0/10     
pods-medium   34s   cpu: 0/50, memory: 0/5Gi, pods: 0/10  
[root@k8s-master apiserver]# kubectl describe quota
Name:       pods-high
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         0     100
memory      0     10Gi
pods        0     10


Name:       pods-low
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         0     10
memory      0     1Gi
pods        0     10


Name:       pods-medium
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         0     50
memory      0     5Gi
pods        0     10
# 启用优先级调度之前需要先创建对应的PriorityClass资源
# 查看PriorityClass的apiVersion, PriorityClass简写为pc
[root@k8s-master apiserver]# kubectl explain pc
KIND:     PriorityClass
VERSION:  scheduling.k8s.io/v1

DESCRIPTION:
     PriorityClass defines mapping from a priority class name to the priority
     integer value. The value can be any valid integer.

FIELDS:
...
# 创建优先级调度指标
[root@k8s-master apiserver]# vi prioritycalss.yml 
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
 name: high
value: 1000000
globalDefault: false
description: "High Priority."
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
 name: medium
value: 10000000
globalDefault: false
description: "Medium Priority."
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
 name: low
value: 100000000
globalDefault: false
description: "Low Priority."
# 创建
[root@k8s-master apiserver]# kubectl apply -f prioritycalss.yml 
priorityclass.scheduling.k8s.io/high created
priorityclass.scheduling.k8s.io/medium created
priorityclass.scheduling.k8s.io/low created
# 查看三个优先级
[root@k8s-master apiserver]# kubectl get pc
NAME                      VALUE        GLOBAL-DEFAULT   AGE
high                      1000000      false            7m32s
low                       100000000    false            7m32s
medium                    10000000     false            7m32s
# 创建Pod, 调用高优先级的指标
[root@k8s-master apiserver]# vi high-priority-pod.yml
apiVersion: v1
kind: Pod
metadata:
  name: high-priority
spec:
  containers:
  - name: high-priority
    image: ubuntu
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello; sleep 10;done"]
    resources:
      requests:
        memory: "1Gi"
        cpu: "50m"
      limits:
        memory: "10Gi"
        cpu: "100m"
  priorityClassName: high
[root@k8s-master apiserver]# kubectl apply -f high-priority-pod.yml 
pod/high-priority created
# 可以看到高优先级的资源配额已经被调度使用了
[root@k8s-master apiserver]# kubectl describe quota
Name:       pods-high
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         50m   100
memory      1Gi   10Gi
pods        1     10


Name:       pods-low
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         0     10
memory      0     1Gi
pods        0     10


Name:       pods-medium
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         0     50
memory      0     5Gi
pods        0     10
# 可以看到default这个Namespace也出现了资源配额的记录
[root@k8s-master apiserver]# kubectl describe ns default
Name:         default
Labels:       <none>
Annotations:  <none>
Status:       Active

Resource Quotas
 Name:     pods-high
 Resource  Used  Hard
 --------  ---   ---
 cpu       50m   100
 memory    1Gi   10Gi
 pods      1     10

 Name:     pods-low
 Resource  Used  Hard
 --------  ---   ---
 cpu       0     10
 memory    0     1Gi
 pods      0     10

 Name:     pods-medium
 Resource  Used  Hard
 --------  ---   ---
 cpu       0     50
 memory    0     5Gi
 pods      0     10

No LimitRange resource.
```

## 8. 基于Namespace

### 8.1. 为namespace配置默认CPU/内存请求和限制

为Namespace=test(测试环境)进行默认的资源限制, 使用LimitRange

```shell
# 切换到测试环境这个Namespace
[root@k8s-master apiserver]# kubens test
Context "kubernetes-admin@kubernetes" modified.
Active namespace is "test".
[root@k8s-master apiserver]# kubens -c
test
# limitrange简写为limits
[root@k8s-master apiserver]# kubectl api-resources | grep limitrange
limitranges                       limits                                      true         LimitRange
# 查看apiVersion和使用描述信息
[root@k8s-master apiserver]# kubectl explain limits
KIND:     LimitRange
VERSION:  v1

DESCRIPTION:
     LimitRange sets resource usage limits for each kind of resource in a
     Namespace.

FIELDS:
...
# 创建limitrange
[root@k8s-master apiserver]# vi limit-range.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limit-range
spec:
  limits:
  - default:
      cpu: 2
      memory: 1024Mi
    defaultRequest:
      cpu: 0.5
      memory: 256Mi
    type: Container
[root@k8s-master apiserver]# kubectl apply -f limit-range.yaml 
limitrange/limit-range created
# 查看约束信息
[root@k8s-master apiserver]# kubectl describe limits limit-range
Name:       limit-range
Namespace:  test
Type        Resource  Min  Max  Default Request  Default Limit  Max Limit/Request Ratio
----        --------  ---  ---  ---------------  -------------  -----------------------
Container   cpu       -    -    500m             2              -
Container   memory    -    -    256Mi            1Gi            -
# 可以看到当前namespace的资源约束情况
[root@k8s-master apiserver]# kubectl describe ns test
Name:         test
Labels:       name=test
Annotations:  <none>
Status:       Active

No resource quota.

Resource Limits
 Type       Resource  Min  Max  Default Request  Default Limit  Max Limit/Request Ratio
 ----       --------  ---  ---  ---------------  -------------  -----------------------
 Container  cpu       -    -    500m             2              -
 Container  memory    -    -    256Mi            1Gi            -
# 尝试突破限制
[root@k8s-master apiserver]# vi limit-pod1.yaml
    imagePullPolicy: Always
apiVersion: v1
kind: Pod
metadata:
  name: limit-pod1
spec:
  containers:
  - name: limit-pod1
    image: nginx
    imagePullPolicy: Always
    resources:
      limits:
        cpu: 3
      requests:
        cpu: 2.1
# 查看pod处于Pending状态
[root@k8s-master apiserver]# kubectl get po
NAME         READY   STATUS    RESTARTS   AGE
limit-pod1   0/1     Pending   0          23s
# 可以看到由于CPU Insufficient(容量不足), 无法创建该Pod, 一直处于Pending
[root@k8s-master apiserver]# kubectl describe po limit-pod1
Name:         limit-pod1
Namespace:    test
...
Status:       Pending
Containers:
  limit-pod1:
    Image:      nginx
    Port:       <none>
    Host Port:  <none>
    Limits:
      cpu:     3
      memory:  1Gi
    Requests:
      cpu:        2100m
      memory:     256Mi
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-hxhq6 (ro)
...
Events:
  Type     Reason            Age               From               Message
  ----     ------            ----              ----               -------
  Warning  FailedScheduling  8s (x3 over 75s)  default-scheduler  0/2 nodes are available: 2 Insufficient cpu.
```

### 8.2. 为namespace配置最大最小CPU/内存约束

```shell
# 先清除之前的限制
[root@k8s-master apiserver]# kubectl delete limits --all
limitrange "limit-range" deleted
[root@k8s-master apiserver]# kubectl describe ns test
Name:         test
Labels:       name=test
Annotations:  <none>
Status:       Active

No resource quota.

No LimitRange resource.
# 创建约束
[root@k8s-master apiserver]# vi limit-range2.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limit-range2
spec:
  limits:
  - max:
      cpu: 1
      memory: 4G
    min:
      cpu: 200m
      memory: 500Mi
    type: Container
[root@k8s-master apiserver]# kubectl apply -f limit-range2.yaml 
limitrange/limit-range2 created
[root@k8s-master apiserver]# kubectl get limits
NAME           CREATED AT
limit-range2   2021-01-07T01:32:09Z
[root@k8s-master apiserver]# kubectl describe ns test
Name:         test
Labels:       name=test
Annotations:  <none>
Status:       Active

No resource quota.

Resource Limits
 Type       Resource  Min    Max  Default Request  Default Limit  Max Limit/Request Ratio
 ----       --------  ---    ---  ---------------  -------------  -----------------------
 Container  cpu       200m   1    1                1              -
 Container  memory    500Mi  4G   4G               4G             -
```

### 8.3. Namespace下Pod配额

限制Namespace下的Pod创建数量

```shell
# 为了直观, 先清除其他限制
[root@k8s-master apiserver]# kubectl delete limits --all
limitrange "limit-range2" deleted
[root@k8s-master apiserver]# kubectl describe ns test
Name:         test
Labels:       name=test
Annotations:  <none>
Status:       Active

No resource quota.

No LimitRange resource.
# 创建Pod配额
[root@k8s-master apiserver]# vi pod-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pod-quota
spec:
  hard:
    pods: 3
[root@k8s-master apiserver]# kubectl apply -f pod-quota.yaml 
resourcequota/pod-quota created
[root@k8s-master apiserver]# kubectl get quota
NAME        AGE   REQUEST     LIMIT
pod-quota   24s   pods: 0/3   
[root@k8s-master apiserver]# kubectl describe ns test
Name:         test
Labels:       name=test
Annotations:  <none>
Status:       Active

Resource Quotas
 Name:     pod-quota
 Resource  Used  Hard
 --------  ---   ---
 pods      0     3

No LimitRange resource.
# 创建deploy, 尝试创建6个副本
[root@k8s-master apiserver]# cat pod-quota-demo.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pod-quota-demo
spec:
  selector:
    matchLabels:
      purpose: quota-demo
  replicas: 6
  template:
    metadata:
      labels:
        purpose: quota-demo
    spec:
      containers:
      - name: pod-quota-demo
        image: nginx
[root@k8s-master apiserver]# kubectl apply -f pod-quota-demo.yaml 
deployment.apps/pod-quota-demo created
[root@k8s-master apiserver]# kubectl get deploy
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
pod-quota-demo   3/6     3            3           60s
# 可见只能创建成功3个, 其他三个无法创建
[root@k8s-master apiserver]# kubectl describe deploy pod-quota-demo
Name:                   pod-quota-demo
Namespace:              test
...
Replicas:               6 desired | 3 updated | 3 total | 3 available | 3 unavailable
...
OldReplicaSets:    <none>
NewReplicaSet:     pod-quota-demo-55bc58fc69 (3/6 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  103s  deployment-controller  Scaled up replica set pod-quota-demo-55bc58fc69 to 6
# 限制已经用完
[root@k8s-master apiserver]# kubectl describe ns test
Name:         test
Labels:       name=test
Annotations:  <none>
Status:       Active

Resource Quotas
 Name:     pod-quota
 Resource  Used  Hard
 --------  ---   ---
 pods      3     3

No LimitRange resource.
```
