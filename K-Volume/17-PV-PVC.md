# Kubernetes存储之PV-PVC

2021-0806

## 概念

为了方便管理存储，PV持久卷子系统为用户和管理员提供了一个API，该API从如何使用存储中抽象出如何提供存储的详细信息。为此，引入了两个新的API资源：PV 和 PVC

PV-PersistentVolume([pərˈsɪstənt] [ˈvɒljuːm]): 持久卷
PVC-PersistentVolumeClaim([pərˈsɪstənt] [ˈvɒljuːm] [kleɪm]): 持久卷声明

PV 是集群中由管理员提供或使用存储类动态提供的一块存储。它是集群中的计算资源。PV是与Volumes类似的卷插件，但**其生命周期与使用PV的任何单个Pod无关**。

PVC 是用户对存储的请求，它类似于Pod。Pods消耗节点资源，PVC消耗PV资源。Pods可以请求特定级别的资源(CPU和内存)。PVC可以请求特定的存储大小和访问模式(如: 读写或只读)

虽然 PVC 允许用户使用抽象的存储资源，但是用户通常需要具有不同属性（比如存储大小等性能）的PV来解决不同的问题。集群管理员需要能够提供各种不同的PV，这些卷在大小和访问模式之外还有很多不同之处，也不向用户公开这些卷是如何实现的细节。

## 生命周期

PV是群集中的资源；PVC是对这些资源的请求，并且还充当对资源的检查。 PV 和 PVC 之间的相互作用遵循以下生命周期：

![20201216094226](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20201216094226.png)

静态资源供应模式下，通过PV和PVC完成绑定，并供Pod使用的存储管理机制

![20201216103258](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20201216103258.png)

动态资源供应模式下，通过StorageClass和PVC完成资源动态绑定（系统自动生成PV），并供Pod使用的存储管理机制

![20201216103404](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20201216103404.png)

### 供应Provisioning

通过集群外的存储系统或者云平台来提供存储持久化支持。两种配置方式：静态或动态

- 静态配置(Static)：集群管理员创建多个PV。它们携带可供集群用户使用的真实存储的详细信息。它们存在于Kubernetes API中可供使用
- 动态配置(Dynamic)：当管理员创建的静态PV均不匹配用户的PVC请求时，集群可能会尝试为PVC动态配置卷。**此配置基于StorageClasses：PVC必须请求一个存储类，并且管理员必须已创建并配置该类才能进行动态配置。** 声明该类为空`""`可有效地禁用其动态配置。要启用基于存储级别的动态存储配置，集群管理员需要启用API Server上的DefaultStorageClass[准入控制器]。例如通过确保DefaultStorageClass位于API Server组件的 `--enable-admission-plugins` 标志, 使用逗号分隔的有序值列表中，可以完成此操作。

### 绑定Binding

用户创建 pvc 并指定需要的资源和访问模式。在找到可用pv之前，pvc会保持未绑定状态。用户创建（或者在动态配置的情况下，已经创建）具有特定存储请求量（大小）和特定访问模式的PVC。主控制器中的控制循环监视新的PV，找到匹配的PV（如果可能的话），并将它们绑定在一起。如果 PV 为新的 PVC 动态配置，那么循环始终将该PV绑定到PVC。否则用户始终至少得到他们所要求的，但是存储量可能会超过所要求的范围。

一旦绑定，无论是如何绑定的，PVC绑定都是互斥的。PVC到PV的绑定是一对一的映射，使用`ClaimRef`。如果不存在匹配的卷，声明(Claims)将无限期保持未绑定，随着匹配量的增加，声明将受到约束。例如配备有许多50Gi PV的群集将与请求100Gi的PVC不匹配。当将100Gi PV添加到群集时，可以绑定PVC。

> **注意：** 静态时PVC与PV绑定时会根据storageClassName(存储类名称)和accessModes(访问模式)判断哪些PV符合绑定需求。然后再根据存储量大小判断，首先存PV储量必须大于或等于PVC声明量；其次就是PV存储量越接近PVC声明量，那么优先级就越高(PV量越小优先级越高)。

### 使用Using

#### 存储使用

集群检查声明以找到绑定卷并为Pod挂载该卷, 对于支持多种访问模式的卷, 用户在其声明中作为Pod中卷使用时指定所需的模式.

一旦用户拥有一个声明并且该声明被绑定, 则绑定的PV就属于该用户, 用户通过在Pod的卷块中包含的PVC部分来调度Pods并访问其声明的PV.

#### PVC保护

使用中的存储对象保护: 该功能的目的是确保在Pod活动时使用的PVC和绑定的PV不会从系统中删除, 防止有用的数据丢失.
如果用户删除了Pod正在使用的PVC, 则不会立即删除该PVC; PVC的清除被推迟, 直到任何Pod不再主动使用PVC, PVC资源的status字段为"termination", 并且其finalizers字段中包含"kubernetes.io/pvc-protection"

> **Tips:** Finalizers(['faɪnəlaɪzər]) 字段属于 Kubernetes GC 垃圾收集器, 是一种删除拦截机制. 常见场景比如: 当删除 namespace 或 pod 等资源时, 有时资源状态会卡在 `Terminating`, 很长时间无法删除, 甚至加上--force 参数之后还是无法正常删除, 这时就需要 edit 该资源, 将 finalizers 字段设置为空, 之后 Kubernetes 资源就正常删除了

```bash
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
# 或者直接注入空数据, 此处以pod为例
[root@k8s-master ~]# kubectl patch pod pod_name -p '{"metadata":{"finalizers":null}}'
```

### 释放Releasing

用户删除pvc来回收存储资源, pv将变成`释放(released)`状态. 由于存储中可能还保留着PVC之前写入的数据, 这些数据需要根据不同的策略来处理, 否则这些存储资源无法被其他pvc使用.

### 回收Recycling

当用户处理完他们的卷时，他们可以从允许回收资源的API中删除PVC对象。PV的回收策略告诉集群在释放卷的声明后该如何处理它。目前，卷可以被保留、回收或删除。

- 保留Retain：保留回收策略允许手动回收资源. 当PVC被删除时, PV仍然存在, 并且该卷被认为是“释放”状态. 由于之前声明的数据仍然存在, 因此另一个声明尚无法得到. 管理员可以手动回收卷
- 删除Delete：对于支持Delete回收策略的卷插件, 删除操作会同时从Kubernetes中删除 PV 对象以及外部基础架构中的关联存储资产, 例如AWS EBS,GCE PD,Azure Disk或Cinder卷. 动态配置的卷将继承其StorageClass的回收策略, 默认为Delete. 管理员应根据用户的期望配置StorageClass
- 回收Recycle：如果基础卷插件支持, Recycle回收策略将`rm -rf /thevolume/*`对该卷执行基本的擦除并使其可用于新的声明

## PV类型

PersistentVolume类型作为插件实现。Kubernetes当前支持以下插件：

```bash
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

### PV卷阶段状态

- Available：尚未绑定的空闲资源
- Bound：卷已经被绑定到claim
- Released：claim被删除，卷处于释放状态，但未被回收
- Failed：卷自动回收失败

```bash
[root@k8s-master pvpvc]# kubectl get pv
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM           STORAGECLASS   REASON   AGE
pv001   2Gi        RWO,RWX        Retain           Available                                           21h
pv002   5Gi        RWO            Retain           Available                                           21h
pv003   20Gi       RWO,RWX        Retain           Available                                           21h
pv004   10Gi       RWO,RWX        Retain           Bound       default/mypvc                           21h
pv005   15Gi       RWO,RWX        Retain           Available                                           21h
```

### PV配置

```bash
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

- Capacity：PV存储容量设置。当前，存储大小是可以设置或请求的唯一资源。将来的属性可能包括IOPS，吞吐量等。
- volumeMode：可选参数，为Filesystem或Block。Filesystem是volumeMode省略参数时使用的默认模式。
- accessModes：访问策略
  - ReadWriteOnce：可以被单个节点以读写方式挂载，命令行中可以被缩写为RWO
  - ReadOnlyMany：可以被许多节点以只读方式挂载，命令行中可以被缩写为ROX
  - ReadWriteMany：可以被多个节点以读写方式挂载，命令行中可以被缩写为RWX
  - ReadWriteOncePod ：只允许被单个Pod访问，需要K8s 1.22+以上版本，并且是CSI创建的PV才可使用
- storageClassName：PV可以有一个类，通过将storageClassName属性设置。特定类的PV只能绑定到请求该类的PVC。
- persistentVolumeReclaimPolicy：回收策略：Retain (保留)-手动回收、Recycle (回收)-基本擦除（rm -rf /thevolume/*）、Delete (删除)-删除相关的存储资产 (例如AWS EBS,GCE PD,Azure Disk或OpenStack Cinder卷).

## PV-PVC实例

### 实例一简单尝试

```bash
# 创建多个pv，存储大小各不相同，访问策略也有差异
[root@k8s-master01 ~]# mkdir -p yamls/volume/pvpvc/
[root@k8s-master01 ~]# cd yamls/volume/pvpvc/
[root@k8s-master01 pvpvc]# vim pvs.yaml
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
[root@k8s-master01 pvpvc]# kubectl apply -f pvs.yaml 
persistentvolume/pv001 created
persistentvolume/pv002 created
persistentvolume/pv003 created
persistentvolume/pv004 created
persistentvolume/pv005 created
[root@k8s-master01 pvpvc]# kubectl get pv -owide
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE   VOLUMEMODE
pv001   2Gi        RWO,RWX        Retain           Available                                   9s    Filesystem
pv002   5Gi        RWO            Retain           Available                                   9s    Filesystem
pv003   20Gi       RWO,RWX        Retain           Available                                   9s    Filesystem
pv004   10Gi       RWO,RWX        Retain           Available                                   9s    Filesystem
pv005   15Gi       RWO,RWX        Retain           Available                                   8s    Filesystem
# 创建PVC，绑定PV
# 存储设为6Gi
[root@k8s-master01 pvpvc]# vim pvc-slow.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mypvc
  namespace: default
spec:
  accessModes: ["ReadWriteMany"]
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
    image: nginx
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html/
[root@k8s-master01 pvpvc]# kubectl apply -f pvc-slow.yaml 
persistentvolumeclaim/mypvc created
pod/vol-pvc created
# 可以看到PVC绑定到了pv004上
[root@k8s-master01 pvpvc]# kubectl get pvc
NAME    STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mypvc   Bound    pv004    10Gi       RWO,RWX                       9s
[root@k8s-master01 pvpvc]# kubectl get pv
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM           STORAGECLASS   REASON   AGE
pv001   2Gi        RWO,RWX        Retain           Available                                           5m3s
pv002   5Gi        RWO            Retain           Available                                           5m3s
pv003   20Gi       RWO,RWX        Retain           Available                                           5m3s
pv004   10Gi       RWO,RWX        Retain           Bound       default/mypvc                           5m3s
pv005   15Gi       RWO,RWX        Retain           Available                                           5m2s
```

### 实例二挂载NFS服务

#### NFS和PV-PVC

NFS服务搭建博客: [Kubernetes K8S存储之搭建NFS](https://www.deemoprobe.com/yunv/k-nfs/)

本次实验已经搭建的NFS服务的IP地址为: 192.168.43.212
NFS服务端(nfs)共享目录是: /data
NFS客户端(k8s-node2)挂载目录是: /mnt

```bash
# 下面操作在NFS服务端执行
# 先清理以前的文件和配置
[root@nfs-server ~]# vim /etc/exports
/data   192.168.43.0/24(rw,sync) # 这行删除
[root@nfs-server ~]# rm -rf /data/client* /data/server*
# 创建pv相关目录
[root@nfs-server ~]# mkdir -p /data/nfs1 /data/nfs2 /data/nfs3 /data/nfs4 /data/nfs5 /data/nfs6
[root@nfs-server ~]# chown -R nfsnobody.nfsnobody /data/ 
[root@nfs-server ~]# ll /data/
total 0
drwxr-xr-x 2 nfsnobody nfsnobody 6 Mar 16 19:08 nfs1
drwxr-xr-x 2 nfsnobody nfsnobody 6 Mar 16 19:08 nfs2
drwxr-xr-x 2 nfsnobody nfsnobody 6 Mar 16 19:08 nfs3
drwxr-xr-x 2 nfsnobody nfsnobody 6 Mar 16 19:08 nfs4
drwxr-xr-x 2 nfsnobody nfsnobody 6 Mar 16 19:08 nfs5
drwxr-xr-x 2 nfsnobody nfsnobody 6 Mar 16 19:08 nfs6
[root@nfs-server ~]# vim /etc/exports
/data/nfs1  192.168.43.0/24(rw,sync,root_squash,all_squash)
/data/nfs2  192.168.43.0/24(rw,sync,root_squash,all_squash)
/data/nfs3  192.168.43.0/24(rw,sync,root_squash,all_squash)
/data/nfs4  192.168.43.0/24(rw,sync,root_squash,all_squash)
/data/nfs5  192.168.43.0/24(rw,sync,root_squash,all_squash)
/data/nfs6  192.168.43.0/24(rw,sync,root_squash,all_squash)
# 我这里nfs和rpcbind已启动, 重启刷新一下配置, 未启动的启动即可
# 先重启rpcbind注册一下nfs配置
[root@nfs-server ~]# systemctl restart rpcbind
[root@nfs-server ~]# systemctl restart nfs
# 查看共享信息
[root@nfs-server ~]# showmount -e 192.168.43.212
Export list for 192.168.43.212:
/data/nfs6 192.168.43.0/24
/data/nfs5 192.168.43.0/24
/data/nfs4 192.168.43.0/24
/data/nfs3 192.168.43.0/24
/data/nfs2 192.168.43.0/24
/data/nfs1 192.168.43.0/24
```

```bash
# 下面操作在nfs客户端k8s-node2上执行
# 查看rpcbind服务状态和NFS可挂载信息
[root@k8s-node02 ~]# systemctl status rpcbind
● rpcbind.service - RPC bind service
   Loaded: loaded (/usr/lib/systemd/system/rpcbind.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2022-03-16 17:59:05 CST; 1h 15min ago
  Process: 767 ExecStart=/sbin/rpcbind -w $RPCBIND_ARGS (code=exited, status=0/SUCCESS)
 Main PID: 783 (rpcbind)
    Tasks: 1
   Memory: 2.0M
   CGroup: /system.slice/rpcbind.service
           └─783 /sbin/rpcbind -w

Mar 16 17:59:04 k8s-node02 systemd[1]: Starting RPC bind service...
Mar 16 17:59:05 k8s-node02 systemd[1]: Started RPC bind service.
[root@k8s-node02 ~]# showmount -e 192.168.43.212
Export list for 192.168.43.212:
/data/nfs6 192.168.43.0/24
/data/nfs5 192.168.43.0/24
/data/nfs4 192.168.43.0/24
/data/nfs3 192.168.43.0/24
/data/nfs2 192.168.43.0/24
/data/nfs1 192.168.43.0/24
```

```bash
# 下面操作在k8s-master01上执行
# 创建PV
[root@k8s-master01 pvpvc]# vim pvs-nfs.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs1
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: nfs
  nfs:
    path: /data/nfs1
    server: 192.168.43.212
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs2
spec:
  capacity:
    storage: 3Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: nfs
  nfs:
    path: /data/nfs2
    server: 192.168.43.212
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs3
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: slow
  nfs:
    path: /data/nfs3
    server: 192.168.43.212
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs4
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: nfs
  nfs:
    path: /data/nfs4
    server: 192.168.43.212
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs5
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: nfs
  nfs:
    path: /data/nfs5
    server: 192.168.43.212
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs6
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: nfs
  nfs:
    path: /data/nfs6
    server: 192.168.43.212
[root@k8s-master01 pvpvc]# kubectl apply -f pvs-nfs.yaml 
persistentvolume/pv-nfs1 created
persistentvolume/pv-nfs2 created
persistentvolume/pv-nfs3 created
persistentvolume/pv-nfs4 created
persistentvolume/pv-nfs5 created
persistentvolume/pv-nfs6 created
# 查看PV信息
[root@k8s-master01 pvpvc]# kubectl get pv -owide | grep pv-nfs -B 1
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM           STORAGECLASS   REASON   AGE   VOLUMEMODE
pv-nfs1   1Gi        RWO            Recycle          Available                   nfs                     83s   Filesystem
pv-nfs2   3Gi        RWO            Recycle          Available                   nfs                     83s   Filesystem
pv-nfs3   5Gi        RWO            Recycle          Available                   slow                    83s   Filesystem
pv-nfs4   10Gi       RWO            Recycle          Available                   nfs                     83s   Filesystem
pv-nfs5   5Gi        RWX            Recycle          Available                   nfs                     83s   Filesystem
pv-nfs6   5Gi        RWO            Recycle          Available                   nfs                     83s   Filesystem
# StatefulSet创建并使用PVC
[root@k8s-master01 pvpvc]# vim sts-pvc.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx
  serviceName: "nginx"
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      terminationGracePeriodSeconds: 100
      nodeSelector: # 匹配到nfs客户端的节点
        nfs: bingo
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "nfs"
      resources:
        requests:
          storage: 3Gi
# 创建
[root@k8s-master01 pvpvc]# kubectl apply -f sts-pvc.yaml 
service/nginx created
statefulset.apps/web created
[root@k8s-master01 pvpvc]# kubectl get svc,sts
NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/nginx        ClusterIP   10.102.17.194   <none>        80/TCP    54s
NAME                   READY   AGE
statefulset.apps/web   3/3     54s
# 查看pod
[root@k8s-master01 pvpvc]# kubectl get po -owide | grep web
web-0                         1/1     Running             0              82s    172.27.14.223    k8s-node02     <none>           <none>
web-1                         1/1     Running             0              72s    172.27.14.224    k8s-node02     <none>           <none>
web-2                         1/1     Running             0              63s    172.27.14.225    k8s-node02     <none>           <none>
# 查看PV和PVC
# 声明了pv-nfs2, pv-nfs4, pv-nfs6
[root@k8s-master01 pvpvc]# kubectl get pv,pvc | grep web
persistentvolume/pv-nfs2   3Gi        RWO            Recycle          Bound       default/www-web-0   nfs                     10m
persistentvolume/pv-nfs4   10Gi       RWO            Recycle          Bound       default/www-web-2   nfs                     10m
persistentvolume/pv-nfs6   5Gi        RWO            Recycle          Bound       default/www-web-1   nfs                     10m
persistentvolumeclaim/www-web-0   Bound    pv-nfs2   3Gi        RWO            nfs            7m40s
persistentvolumeclaim/www-web-1   Bound    pv-nfs6   5Gi        RWO            nfs            7m24s
persistentvolumeclaim/www-web-2   Bound    pv-nfs4   10Gi       RWO            nfs            118s
```

PVC与PV绑定时会根据storageClassName（存储类名称）和accessModes（访问模式）判断哪些PV符合绑定需求。然后再根据存储量大小判断，首先存PV储量必须大于或等于PVC声明量；其次就是PV存储量越接近PVC声明量，那么优先级就越高（PV量越小优先级越高）。

#### 验证

在NFS服务端对应NFS共享目录创建文件

```bash
[root@nfs-server ~]# cd /data/
[root@nfs-server data]# cd nfs2/
[root@nfs-server nfs2]# echo "pv-nfs2" >> index.html
[root@nfs-server nfs2]# cd ../nfs4/
[root@nfs-server nfs4]# echo "pv-nfs4" >> index.html
[root@nfs-server nfs4]# cd ../nfs6/
[root@nfs-server nfs6]# echo "pv-nfs6" >> index.html
```

curl访问pod

```bash
[root@k8s-master01 pvpvc]# kubectl get po -owide | grep web
web-0                         1/1     Running             0              82s    172.27.14.223    k8s-node02     <none>           <none>
web-1                         1/1     Running             0              72s    172.27.14.224    k8s-node02     <none>           <none>
web-2                         1/1     Running             0              63s    172.27.14.225    k8s-node02     <none>           <none>
[root@k8s-master01 pvpvc]# curl 172.27.14.223 
pv-nfs2
[root@k8s-master01 pvpvc]# curl 172.27.14.224 
pv-nfs6
[root@k8s-master01 pvpvc]# curl 172.27.14.225
pv-nfs4
```

#### 删除sts并回收PV

```bash
# 删除statefulset
[root@k8s-master01 pvpvc]# kubectl delete -f sts-pvc.yaml 
service "nginx" deleted
statefulset.apps "web" deleted

# 查看PVC和PV,并删除PVC
[root@k8s-master01 pvpvc]# kubectl get pvc -owide | grep web
www-web-0   Bound    pv-nfs2   3Gi        RWO            nfs            12m     Filesystem
www-web-1   Bound    pv-nfs6   5Gi        RWO            nfs            12m     Filesystem
www-web-2   Bound    pv-nfs4   10Gi       RWO            nfs            6m41s   Filesystem
[root@k8s-master01 pvpvc]# kubectl get pv -owide | grep nfs
pv-nfs1   1Gi        RWO            Recycle          Available                       nfs                     15m   Filesystem
pv-nfs2   3Gi        RWO            Recycle          Bound       default/www-web-0   nfs                     15m   Filesystem
pv-nfs3   5Gi        RWO            Recycle          Available                       slow                    15m   Filesystem
pv-nfs4   10Gi       RWO            Recycle          Bound       default/www-web-2   nfs                     15m   Filesystem
pv-nfs5   5Gi        RWX            Recycle          Available                       nfs                     15m   Filesystem
pv-nfs6   5Gi        RWO            Recycle          Bound       default/www-web-1   nfs                     15m   Filesystem
[root@k8s-master01 pvpvc]# kubectl delete pvc www-web-0 www-web-1 www-web-2 
persistentvolumeclaim "www-web-0" deleted
persistentvolumeclaim "www-web-1" deleted
persistentvolumeclaim "www-web-2" deleted
# PV声明删除后, 会自动回收PV, 但PV需要检测是否还有其他声明在使用所以不会立即删除PV, 需要等一段时间, 几分钟不等
# 中间状态
[root@k8s-master01 pvpvc]# kubectl get pv -owide | grep nfs
pv-nfs1   1Gi        RWO            Recycle          Available                       nfs                     16m   Filesystem
pv-nfs2   3Gi        RWO            Recycle          Released    default/www-web-0   nfs                     16m   Filesystem
pv-nfs3   5Gi        RWO            Recycle          Available                       slow                    16m   Filesystem
pv-nfs4   10Gi       RWO            Recycle          Released    default/www-web-2   nfs                     16m   Filesystem
pv-nfs5   5Gi        RWX            Recycle          Available                       nfs                     16m   Filesystem
pv-nfs6   5Gi        RWO            Recycle          Released    default/www-web-1   nfs                     16m   Filesystem
# 等了很长时间, 发现pv-nfs6还没有回收, 可以选择继续等待自动回收或者手动回收
# 编辑pv-nfs6
[root@k8s-master01 pvpvc]# kubectl edit pv pv-nfs6
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
kind: PersistentVolume
metadata:
...
  # 可以看到该PV处于保护状态
  finalizers:
  - kubernetes.io/pv-protection
  name: pv-nfs6
  resourceVersion: "464943"
  selfLink: /api/v1/persistentvolumes/pv-nfs6
  uid: 1bd446ec-a3ba-4455-9e4d-6c1711c6ed7e
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 5Gi
  ###删除以下7行###
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: www-web-1
    namespace: default
    resourceVersion: "463514"
    uid: 130a922c-d468-4f0a-a701-ca124f0619b3
  ###删除以上7行###
  nfs:
    path: /data/nfs6
    server: 192.168.43.212
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: nfs
  volumeMode: Filesystem
status:
  phase: Released
# 清除claimRef配置后, 查看PV, 释放成功
# 最终状态
[root@k8s-master01 pvpvc]# kubectl get pv -owide | grep nfs
pv-nfs1   1Gi        RWO            Recycle          Available                   nfs                     16m   Filesystem
pv-nfs2   3Gi        RWO            Recycle          Available                   nfs                     16m   Filesystem
pv-nfs3   5Gi        RWO            Recycle          Available                   slow                    16m   Filesystem
pv-nfs4   10Gi       RWO            Recycle          Available                   nfs                     16m   Filesystem
pv-nfs5   5Gi        RWX            Recycle          Available                   nfs                     16m   Filesystem
pv-nfs6   5Gi        RWO            Recycle          Available                   nfs                     16m   Filesystem
# 可以查看一下nfs服务端共享文件也都自动删除了
[root@nfs-server data]# ll nfs2/
total 0
[root@nfs-server data]# ll nfs4/
total 0
[root@nfs-server data]# ll nfs6/
total 0
```

#### STS与PVC

- 匹配StatefulSet的Pod name(网络标识)的模式为：$(statefulset名称)-$(序号),比如StatefulSet名称为web，副本数为3则为：web-0、web-1、web-2
- StatefulSet为每个Pod副本创建了一个DNS域名，这个域名的格式为：`$(podname).(headless service name)`，也就意味着服务之间是通过Pod域名来通信而非Pod IP。当Pod所在Node发生故障时，Pod会被漂移到其他Node上，Pod IP会发生改变，但Pod域名不会变化
- StatefulSet使用Headless服务来控制Pod的域名，这个Headless服务域名的为：`$(service name).$(namespace).svc.cluster.local`，其中 cluster.local 指定的集群的域名
- 根据volumeClaimTemplates，为每个Pod创建一个PVC，PVC的命令规则为：`$(volumeClaimTemplates name)-$(pod name)`，比如volumeClaimTemplates为www,pod name为web-0、web-1、web-2；那么创建出来的PVC为：www-web-0、www-web-1、www-web-2
- 删除Pod不会删除对应的PVC，手动删除PVC将自动释放PV

### 异常状态可能原因

PVC一直Pending的原因：

- PVC的空间申请大小大于PV的大小
- PVC的StorageClassName没有和PV的一致
- PVC的accessModes和PV的不一致

挂载PVC的Pod一直处于Pending：

- PVC没有创建成功/PVC不存在
- PVC和Pod不在同一个Namespace
