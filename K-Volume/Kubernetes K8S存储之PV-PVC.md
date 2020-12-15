# 1. Kubernetes K8S存储之PV-PVC

为了方便管理存储, PersistentVolume子系统为用户和管理员提供了一个API, 该API从如何使用存储中抽象出如何提供存储的详细信息.为此, 引入了两个新的API资源：PersistentVolume(PV)和PersistentVolumeClaim(PVC)

## 1.1. PV概述

PersistentVolume (PV)是集群中由管理员提供或使用存储类动态提供的一块存储。它是集群中的资源，就像节点是集群资源一样

PV是与Volumes类似的卷插件，但其生命周期与使用PV的任何单个Pod无关。由此API对象捕获存储的实现细节，不管是NFS、iSCSI还是特定于云提供商的存储系统

## 1.2. PVC概述

PersistentVolumeClaim (PVC) 是用户对存储的请求。它类似于Pod；Pods消耗节点资源，而PVC消耗PV资源。Pods可以请求特定级别的资源(CPU和内存)。Claim可以请求特定的存储大小和访问模式(例如，它们可以挂载一次读写或多次只读)

虽然PersistentVolumeClaims (PVC) 允许用户使用抽象的存储资源，但是用户通常需要具有不同属性(比如性能)的PersistentVolumes (PV) 来解决不同的问题。集群管理员需要能够提供各种不同的PersistentVolumes，这些卷在大小和访问模式之外还有很多不同之处，也不向用户公开这些卷是如何实现的细节。对于这些需求，有一个StorageClass资源

## 1.3. 生命周期

PV是群集中的资源。PVC是对这些资源的请求，并且还充当对资源的检查。PV和PVC之间的相互作用遵循以下生命周期：

Provisioning ——-> Binding ——–>Using——>Releasing——>Recycling

- 供应准备Provisioning---通过集群外的存储系统或者云平台来提供存储持久化支持。
  - 静态提供Static：集群管理员创建多个PV。 它们携带可供集群用户使用的真实存储的详细信息。 它们存在于Kubernetes API中，可用于消费
  - 动态提供Dynamic：当管理员创建的静态PV都不匹配用户的PersistentVolumeClaim时，集群可能会尝试为PVC动态配置卷。 此配置基于StorageClasses：PVC必须请求一个类，并且管理员必须已创建并配置该类才能进行动态配置。 要求该类的声明有效地为自己禁用动态配置。

- 绑定Binding---用户创建pvc并指定需要的资源和访问模式。在找到可用pv之前，pvc会保持未绑定状态。
- 使用Using---用户可在pod中像volume一样使用pvc。
- 释放Releasing---用户删除pvc来回收存储资源，pv将变成“released”状态。由于还保留着之前的数据，这些数据需要根据不同的策略来处理，否则这些存储资源无法被其他pvc使用。
- 回收Recycling---pv可以设置三种回收策略：保留（Retain），回收（Recycle）和删除（Delete）
  - 保留策略：允许人工处理保留的数据。
  - 删除策略：将删除pv和外部关联的存储资源，需要插件支持。
  - 回收策略：将执行清除操作，之后可以被新的pvc使用，需要插件支持。

注：目前只有NFS和HostPath类型卷支持回收策略，AWS EBS,GCE PD,Azure Disk和Cinder支持删除(Delete)策略

## 1.4. Persistent Volumes类型

PersistentVolume类型作为插件实现。Kubernetes当前支持以下插件：

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
HostPath (仅用于单节点测试——本地存储不受任何方式的支持，也不能在多节点集群中工作)
Portworx Volumes
ScaleIO Volumes
StorageOS
```

### 1.4.1. PV卷阶段状态

- Available – 资源尚未被claim使用
- Bound – 卷已经被绑定到claim了
- Released – claim被删除，卷处于释放状态，但未被集群回收。
- Failed – 卷自动回收失败

### 1.4.2. 配置文件模板

```shell
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003
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

- Capacity：通常，PV将具有特定的存储容量设置。当前，存储大小是可以设置或请求的唯一资源。将来的属性可能包括IOPS，吞吐量等。
- volumeMode：可选参数，为Filesystem或Block。Filesystem是volumeMode省略参数时使用的默认模式。
- accessModes：PersistentVolume可以通过资源提供者支持的任何方式安装在主机上。如下文表中所示，提供商将具有不同的功能，并且每个PV的访问模式都将设置为该特定卷支持的特定模式。例如，NFS可以支持多个读/写客户端，但是特定的NFS PV可能以只读方式在服务器上导出。每个PV都有自己的一组访问模式，用于描述该特定PV的功能。
  - ReadWriteOnce-该卷可以被单个节点以读写方式挂载
  - ReadOnlyMany-该卷可以被许多节点以只读方式挂载
  - ReadWriteMany-该卷可以被多个节点以读写方式挂载
- storageClassName：PV可以有一个类，通过将storageClassName属性设置为一个StorageClass的名称来指定这个类。特定类的PV只能绑定到请求该类的PVC。没有storageClassName的PV没有类，只能绑定到不请求特定类的PVC。
- persistentVolumeReclaimPolicy：当前的回收政策是：Retain (保留)-手动回收、Recycle (回收)-基本擦除（rm -rf /thevolume/*）、Delete (删除)-删除相关的存储资产 (例如AWS EBS，GCE PD，Azure Disk或OpenStack Cinder卷)。

备注：当前，仅NFS和HostPath支持回收。AWS EBS，GCE PD，Azure Disk和Cinder卷支持删除

## 1.5. PV-PVC示例

```shell
# 创建多个pv，存储大小各不相同，是否可读也不相同
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
# 创建PVC，绑定PV
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
