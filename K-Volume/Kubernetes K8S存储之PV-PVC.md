# Kubernetes K8S存储之PV-PVC

为了方便管理存储, PV持久卷子系统为用户和管理员提供了一个API, 该API从如何使用存储中抽象出如何提供存储的详细信息.为此, 引入了两个新的API资源: PV 和 PVC

PV-PersistentVolume([pərˈsɪstənt] [ˈvɒljuːm]): 持久卷
PVC-PersistentVolumeClaim([pərˈsɪstənt] [ˈvɒljuːm] [kleɪm]): 持久卷声明

## 1. PV概述

PV 是集群中由管理员提供或使用存储类动态提供的一块存储. 它是集群中的资源, 就像节点是集群资源一样

PV是与Volumes类似的卷插件, 但**其生命周期与使用PV的任何单个Pod无关**.由此API对象捕获存储的实现细节,不管是NFS、iSCSI还是特定于云提供商的存储系统

## 2. PVC概述

PVC 是用户对存储的请求, 它类似于Pod; Pods消耗节点资源,而PVC消耗PV资源. Pods可以请求特定级别的资源(CPU和内存). Claim可以请求特定的存储大小和访问模式(例如: 它们可以挂载一次读写或多次只读)

虽然 PVC 允许用户使用抽象的存储资源, 但是用户通常需要具有不同属性(比如存储大小等性能)的 PV 来解决不同的问题. 集群管理员需要能够提供各种不同的PV, 这些卷在大小和访问模式之外还有很多不同之处, 也不向用户公开这些卷是如何实现的细节.

## 3. 生命周期

PV是群集中的资源; PVC是对这些资源的请求,并且还充当对资源的检查. PV 和 PVC 之间的相互作用遵循以下生命周期:

![20201216094226](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20201216094226.png)

图1: 静态资源供应模式下, 通过PV和PVC完成绑定, 并供Pod使用的存储管理机制

![20201216103258](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20201216103258.png)

图2: 动态资源供应模式下, 通过StorageClass和PVC完成资源动态绑定(系统自动生成PV), 并供Pod使用的存储管理机制

![20201216103404](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20201216103404.png)

### 3.1. 供应Provisioning

通过集群外的存储系统或者云平台来提供存储持久化支持. 两种配置方式: 静态或动态

- 静态配置(Static): 集群管理员创建多个PV. 它们携带可供集群用户使用的真实存储的详细信息. 它们存在于Kubernetes API中,可供使用
- 动态配置(Dynamic): 当管理员创建的静态PV均不匹配用户的PVC请求时, 集群可能会尝试为PVC动态配置卷. **此配置基于StorageClasses:PVC必须请求一个存储类, 并且管理员必须已创建并配置该类才能进行动态配置.** 声明该类为`""`, 可有效地禁用其动态配置.要启用基于存储级别的动态存储配置,集群管理员需要启用API Server上的DefaultStorageClass[准入控制器].例如, 通过确保DefaultStorageClass位于API Server组件的 `--enable-admission-plugins` 标志, 使用逗号分隔的有序值列表中, 可以完成此操作.

### 3.2. 绑定Binding

用户创建 pvc 并指定需要的资源和访问模式. 在找到可用pv之前,pvc会保持未绑定状态. 用户创建(或者在动态配置的情况下,已经创建)具有特定存储请求量(大小)和特定访问模式的PVC. 主控制器中的控制循环监视新的PV, 找到匹配的PV(如果可能的话), 并将它们绑定在一起. 如果 PV 为新的 PVC 动态配置, 那么循环始终将该PV绑定到PVC. 否则, 用户始终至少得到他们所要求的, 但是存储量可能会超过所要求的范围.

一旦绑定, 无论是如何绑定的, PVC绑定都是互斥的. PVC到PV的绑定是一对一的映射, 使用ClaimRef, 它是PV和PVC之间的双向绑定.

如果不存在匹配的卷, 声明(Claims)将无限期保持未绑定, 随着匹配量的增加, 声明将受到约束. 例如,配备有许多50Gi PV的群集将与请求100Gi的PVC不匹配. 当将100Gi PV添加到群集时,可以绑定PVC.

> **注意:** 静态时PVC与PV绑定时会根据storageClassName(存储类名称)和accessModes(访问模式)判断哪些PV符合绑定需求. 然后再根据存储量大小判断, 首先存PV储量必须大于或等于PVC声明量; 其次就是PV存储量越接近PVC声明量, 那么优先级就越高(PV量越小优先级越高).

### 3.3. 使用Using

#### 3.3.1. 存储使用

集群检查声明以找到绑定卷并为Pod挂载该卷, 对于支持多种访问模式的卷, 用户在其声明中作为Pod中卷使用时指定所需的模式.

一旦用户拥有一个声明并且该声明被绑定, 则绑定的PV就属于该用户, 用户通过在Pod的卷块中包含的PVC部分来调度Pods并访问其声明的PV.

#### 3.3.2. PVC保护

使用中的存储对象保护: 该功能的目的是确保在Pod活动时使用的PVC和绑定的PV不会从系统中删除, 防止有用的数据丢失.
如果用户删除了Pod正在使用的PVC, 则不会立即删除该PVC; PVC的清除被推迟, 直到任何Pod不再主动使用PVC, PVC资源的status字段为"termination", 并且其finalizers字段中包含"kubernetes.io/pvc-protection"

> **Tips:** Finalizers(['faɪnəlaɪzər]) 字段属于 Kubernetes GC 垃圾收集器, 是一种删除拦截机制. 常见场景比如: 当删除 namespace 或 pod 等资源时, 有时资源状态会卡在 `Terminating`, 很长时间无法删除, 甚至加上--force 参数之后还是无法正常删除, 这时就需要 edit 该资源, 将 finalizers 字段设置为空, 之后 Kubernetes 资源就正常删除了

```shell
[root@k8s-master ~]# kubectl edit ns kube-public
# Please edit the object below. Lines beginning with a '#' will be ignored,
apiVersion: v1
kind: Namespace
...
spec:
# Finalizers字段
  finalizers:
  - kubernetes
...
# 如果是Pod等资源, edit对应资源即可
[root@k8s-master ~]# kubectl edit po pod_name
```

### 3.4. 释放Releasing

用户删除pvc来回收存储资源, pv将变成`释放(released)`状态. 由于存储中可能还保留着PVC之前写入的数据, 这些数据需要根据不同的策略来处理, 否则这些存储资源无法被其他pvc使用.

### 3.5. 回收Recycling

当用户处理完他们的卷时, 他们可以从允许回收资源的API中删除PVC对象. PV的回收策略告诉集群在释放卷的声明后该如何处理它.目前,卷可以被保留、回收或删除.

#### 3.5.1. Retain (保留)

保留回收策略允许手动回收资源. 当PVC被删除时, PV仍然存在, 并且该卷被认为是“释放”状态. 由于之前声明的数据仍然存在, 因此另一个声明尚无法得到. 管理员可以手动回收卷.

#### 3.5.2. Delete (删除)

对于支持Delete回收策略的卷插件, 删除操作会同时从Kubernetes中删除 PV 对象以及外部基础架构中的关联存储资产, 例如AWS EBS,GCE PD,Azure Disk或Cinder卷. 动态配置的卷将继承其StorageClass的回收策略, 默认为Delete. 管理员应根据用户的期望配置StorageClass.

#### 3.5.3. Recycle (回收)

如果基础卷插件支持, Recycle回收策略将rm -rf /thevolume/*对该卷执行基本的擦除并使其可用于新的声明.

## 4. PV类型

PersistentVolume类型作为插件实现.Kubernetes当前支持以下插件:

```shell
GCEPersistentDisk
AWSElasticBlockStore
AzureFile
AzureDisk
CSI
FC (Fibre Channel)
FlexVolume
Flocker
NFS
iSCSI
RBD (Ceph Block Device)
CephFS
Cinder (OpenStack block storage)
Glusterfs
VsphereVolume
Quobyte Volumes
HostPath (仅用于单节点测试——本地存储不受任何方式的支持,也不能在多节点集群中工作)
Portworx Volumes
ScaleIO Volumes
StorageOS
```

### 4.1. PV卷阶段状态

- Available – 资源尚未被claim使用
- Bound – 卷已经被绑定到claim了
- Released – claim被删除,卷处于释放状态,但未被集群回收.
- Failed – 卷自动回收失败

```shell
[root@k8s-master pvpvc]# kubectl get pv
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM           STORAGECLASS   REASON   AGE
pv001   2Gi        RWO,RWX        Retain           Available                                           21h
pv002   5Gi        RWO            Retain           Available                                           21h
pv003   20Gi       RWO,RWX        Retain           Available                                           21h
pv004   10Gi       RWO,RWX        Retain           Bound       default/mypvc                           21h
pv005   15Gi       RWO,RWX        Retain           Available                                           21h
```

### 4.2. 配置文件样例

```shell
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv001
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: slow
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /tmp
    server: 172.17.0.2
```

- Capacity: 通常,PV将具有特定的存储容量设置.当前,存储大小是可以设置或请求的唯一资源.将来的属性可能包括IOPS,吞吐量等.
- volumeMode: 可选参数,为Filesystem或Block.Filesystem是volumeMode省略参数时使用的默认模式.
- accessModes: PersistentVolume可以通过资源提供者支持的任何方式安装在主机上.如下文表中所示,提供商将具有不同的功能,并且每个PV的访问模式都将设置为该特定卷支持的特定模式.例如,NFS可以支持多个读/写客户端,但是特定的NFS PV可能以只读方式在服务器上导出.每个PV都有自己的一组访问模式,用于描述该特定PV的功能.
  - ReadWriteOnce-该卷可以被单个节点以读写方式挂载
  - ReadOnlyMany-该卷可以被许多节点以只读方式挂载
  - ReadWriteMany-该卷可以被多个节点以读写方式挂载
- storageClassName: PV可以有一个类,通过将storageClassName属性设置为一个StorageClass的名称来指定这个类.特定类的PV只能绑定到请求该类的PVC.没有storageClassName的PV没有类,只能绑定到不请求特定类的PVC.
- persistentVolumeReclaimPolicy: 当前的回收政策是: Retain (保留)-手动回收、Recycle (回收)-基本擦除（rm -rf /thevolume/*）、Delete (删除)-删除相关的存储资产 (例如AWS EBS,GCE PD,Azure Disk或OpenStack Cinder卷).

备注: 当前,仅NFS和HostPath支持回收.AWS EBS,GCE PD,Azure Disk和Cinder卷支持删除

## 5. PV-PVC示例

```shell
# 创建多个pv,存储大小各不相同,是否可读也不相同
[root@k8s-master pvpvc]# vi pv_demo.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv001
  labels:
    name: pv001
spec:
  nfs:
    path: /data/volumes/v1
    server: nfs
  accessModes: ["ReadWriteMany","ReadWriteOnce"]
  capacity:
    storage: 2Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv002
  labels:
    name: pv002
spec:
  nfs:
    path: /data/volumes/v2
    server: nfs
  accessModes: ["ReadWriteOnce"]
  capacity:
    storage: 5Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv003
  labels:
    name: pv003
spec:
  nfs:
    path: /data/volumes/v3
    server: nfs
  accessModes: ["ReadWriteMany","ReadWriteOnce"]
  capacity:
    storage: 20Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv004
  labels:
    name: pv004
spec:
  nfs:
    path: /data/volumes/v4
    server: nfs
  accessModes: ["ReadWriteMany","ReadWriteOnce"]
  capacity:
    storage: 10Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv005
  labels:
    name: pv005
spec:
  nfs:
    path: /data/volumes/v5
    server: nfs
  accessModes: ["ReadWriteMany","ReadWriteOnce"]
  capacity:
    storage: 15Gi
"pv_demo.yaml" [New] 69L, 1113C written
[root@k8s-master pvpvc]# kubectl apply -f pv_demo.yaml 
persistentvolume/pv001 created
persistentvolume/pv002 created
persistentvolume/pv003 created
persistentvolume/pv004 created
persistentvolume/pv005 created
# 查看PV, 
[root@k8s-master pvpvc]# kubectl get pv
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
pv001   2Gi        RWO,RWX        Retain           Available                                   56s
pv002   5Gi        RWO            Retain           Available                                   56s
pv003   20Gi       RWO,RWX        Retain           Available                                   56s
pv004   10Gi       RWO,RWX        Retain           Available                                   56s
pv005   15Gi       RWO,RWX        Retain           Available                                   56s
# 创建PVC,绑定PV
# 存储设为6Gi
[root@k8s-master pvpvc]# vi pvc-demo.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mypvc
  namespace: default
spec:
  accessModes: ["ReadWriteMany"]
  storageClassName: slow
  resources:
    requests:
      storage: 6Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: vol-pvc
  namespace: default
spec:
  volumes:
  - name: html
    persistentVolumeClaim:
      claimName: mypvc
  containers:
  - name: myapp
    image: ikubernetes/myapp:v1
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html/
[root@k8s-master pvpvc]# kubectl apply -f pvc-demo.yaml 
persistentvolumeclaim/mypvc created
pod/vol-pvc created
# 可以看到PVC绑定到了pv004上
[root@k8s-master pvpvc]# kubectl get pvc
NAME    STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mypvc   Bound    pv004    10Gi       RWO,RWX                       14s
[root@k8s-master pvpvc]# kubectl get pv
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM           STORAGECLASS   REASON   AGE
pv001   2Gi        RWO,RWX        Retain           Available                                           4m52s
pv002   5Gi        RWO            Retain           Available                                           4m52s
pv003   20Gi       RWO,RWX        Retain           Available                                           4m52s
pv004   10Gi       RWO,RWX        Retain           Bound       default/mypvc                           4m52s
pv005   15Gi       RWO,RWX        Retain           Available                                           4m52s
```
