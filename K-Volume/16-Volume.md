# Kubernetes存储之Volume

2021-08-04

## 概念

在容器中的文件在磁盘上是临时存放的，当容器关闭时这些临时文件也会被一并清除。这给容器中运行的特殊应用程序带来一些问题。

首先，当容器崩溃时，kubelet将重新启动容器，容器中的文件将会丢失——因为容器会以干净的状态重建。其次，当在一个 Pod 中同时运行多个容器时，常常需要在这些容器之间共享文件.Kubernetes 抽象出Volume对象来解决这类问题。

Kubernetes Volume卷具有明确的生命周期（与挂载它的 Pod 相同）。因此，Volume比Pod中运行的任何容器的存活期都长，在容器重新启动时数据也会得到保留。当然，当一个Pod不再存在时，Volume也将不再存在。更重要的是，Kubernetes可以支持许多类型的Volume卷，Pod 也能同时使用任意数量的Volume卷。

使用volume时，Pod声明中需要提供卷的类型`.spec.volumes`字段和卷挂载的位置`.spec.containers.volumeMounts`字段

## Volume类型

目前Kubernetes支持以下类型的Volume，详情解释请见[官网: Kubernetes卷说明](https://kubernetes.io/zh/docs/concepts/storage/volumes/)

```bash
awsElasticBlockStore
azureDisk
azureFile
cephfs
cinder
configMap
csi
downwardAPI
emptyDir
fc (光纤通道)
flexVolume
gcePersistentDisk
glusterfs
hostPath
iscsi
local
nfs
persistentVolumeClaim
projected
portworxVolume
quobyte
rbd
scaleIO
secret
storageos
vsphereVolume
```

常用的类型主要有: emptyDir、hostPath、Secret和ConfigMap

## emptyDir

当Pod指定到某个节点上时，首先创建的是一个emptyDir卷，并且只要Pod在该节点上运行，卷就一直存在。就像它的名称表示的那样，卷最初是空的。尽管Pod中每个容器挂载emptyDir卷的路径可能相同也可能不同，但是这些容器都可以读写emptyDir卷中相同的文件。如果Pod中有多个容器，其中某个容器重启，不会影响emptyDir卷中的数据。当 Pod 因为某些原因被删除时，emptyDir 卷中的数据也会永久删除。

注意：容器崩溃并不会导致Pod被从节点上移除，因此容器崩溃时emptyDir卷中的数据是安全的。

emptyDir用途

- 缓存空间，例如基于磁盘的归并排序
- 为耗时较长的计算任务提供检查点，以便任务能方便地从崩溃前状态恢复执行
- 在Web服务器容器服务数据时，保存内容管理器容器获取的文件

### emptyDir示例

```bash
[root@k8s-master01 ~]# mkdir yamls/volume/emptydir -p
[root@k8s-master01 ~]# cd yamls/volume/emptydir/
[root@k8s-master01 emptydir]# vim pod-emptydir.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-emptydir
  namespace: default
spec:
  containers:
  - name: myapp-pod
    image: registry.cn-beijing.aliyuncs.com/google_registry/myapp:v1
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  - name: busybox-pod
    image: registry.cn-beijing.aliyuncs.com/google_registry/busybox:1.24
    imagePullPolicy: IfNotPresent
    command: ["/bin/sh", "-c", "sleep 3600"]
    volumeMounts:
    - mountPath: /test/cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir: {}
[root@k8s-master01 emptydir]# kubectl apply -f pod-emptydir.yaml 
pod/pod-emptydir created
[root@k8s-master01 emptydir]# kubectl get po pod-emptydir -owide
NAME           READY   STATUS    RESTARTS   AGE   IP              NODE           NOMINATED NODE   READINESS GATES
pod-emptydir   2/2     Running   0          23s   172.18.195.32   k8s-master03   <none>           <none>
[root@k8s-master01 emptydir]# kubectl describe po pod-emptydir 
Name:         pod-emptydir
Namespace:    default
Priority:     0
Node:         k8s-master03/192.168.43.185
Start Time:   Wed, 16 Mar 2022 17:14:47 +0800
Labels:       <none>
Annotations:  cni.projectcalico.org/containerID: 7e261b1868fbe140d2aa0277c1a5f5dc8b820a14042e6f612c3ddaeb77fbefe5
              cni.projectcalico.org/podIP: 172.18.195.32/32
              cni.projectcalico.org/podIPs: 172.18.195.32/32
Status:       Running
IP:           172.18.195.32
IPs:
  IP:  172.18.195.32
Containers:
  myapp-pod:
    Container ID:   containerd://25664b45e687a97527697085777a1df103297b1dd6e986a955f6fbfad472a124
    Image:          registry.cn-beijing.aliyuncs.com/google_registry/myapp:v1
    Image ID:       registry.cn-beijing.aliyuncs.com/google_registry/myapp@sha256:9eeca44ba2d410e54fccc54cbe9c021802aa8b9836a0bcf3d3229354e4c8870e
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Wed, 16 Mar 2022 17:14:48 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /cache from cache-volume (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-48rl2 (ro)
  busybox-pod:
    Container ID:  containerd://ffa1324b2f2afe373ee1de720fb926c93e719d8d078d1e1d0aa01cc772190bd7
    Image:         registry.cn-beijing.aliyuncs.com/google_registry/busybox:1.24
    Image ID:      registry.cn-beijing.aliyuncs.com/google_registry/busybox@sha256:f73ae051fae52945d92ee20d62c315306c593c59a429ccbbdcba4a488ee12269
    Port:          <none>
    Host Port:     <none>
    Command:
      /bin/sh
      -c
      sleep 3600
    State:          Running
      Started:      Wed, 16 Mar 2022 17:14:58 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /test/cache from cache-volume (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-48rl2 (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
  ...
```

创建了`myapp-pod`和`busybox-pod`两个容器

### emptyDir验证

```bash
# 进入myapp-pod容器
[root@k8s-master01 emptydir]# kubectl exec -it pod-emptydir -c myapp-pod -- sh
/ # cd /cache/
/cache # echo "myapp-pod" > myapp-pod
/cache # cat myapp-pod 
myapp-pod
/cache # 
# 进入busybox-pod容器
[root@k8s-master01 emptydir]# kubectl exec -it pod-emptydir -c busybox-pod -- sh
/ # cd /test/cache/
/test/cache # ls
myapp-pod
/test/cache # cat myapp-pod 
myapp-pod
/test/cache # echo busybox-pod >> myapp-pod 
/test/cache # cat myapp-pod 
myapp-pod
busybox-pod
/test/cache #
```

由上可见,一个Pod中多个容器可共享同一个emptyDir卷

## hostPath

hostPath 卷能将主机node节点文件系统上的文件或目录挂载到你的 Pod 中。

- 运行一个需要访问 Docker 引擎内部机制的容器；请使用 hostPath 挂载 /var/lib/docker 路径
- 在容器中运行 cAdvisor 时，以 hostPath 方式挂载 /sys
- 允许 Pod 指定给定的 hostPath 在运行 Pod 之前是否应该存在，是否应该创建以及应该以什么方式存在

### 支持类型

除了必需的 path 属性之外，用户可以选择性地为 hostPath 卷指定 type。支持的 type 值如下：

| Type取值          | 描述                                                                                                                      |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------- |
|                   | 空字符串（默认）用于向后兼容，这意味着在挂载hostPath卷之前不会执行任何检查                                                |
| DirectoryOrCreate | 如果指定的路径不存在，那么将根据需要创建空目录，权限设置为0755，具有与 Kubelet 相同的组和所有权                           |
| Directory         | 给定的路径必须存在                                                                                                        |
| FileOrCreate      | 如果给定路径的文件不存在，那么根据需要创建空文件，权限为0644，具有与Kubelet相同的组和所有权（前提: 文件所在目录必须存在） |
| File              | 给定路径上的文件必须存在                                                                                                  |
| Socket            | 在给定路径上必须存在的 UNIX 套接字                                                                                        |
| CharDevice        | 在给定路径上必须存在的字符设备                                                                                            |
| BlockDevice       | 在给定路径上必须存在的块设备                                                                                              |

注意事项：

- 具有相同配置（例如从 podTemplate 创建）的多个 Pod 会由于节点上文件的不同而在不同节点上有不同的行为
- 当 Kubernetes 按照计划添加资源感知的调度时，这类调度机制将无法考虑由 hostPath 卷使用的资源
- 基础主机上创建的文件或目录只能由 root 用户写入。需要在特权容器中以root身份运行进程，或者修改主机上的文件权限以便容器能够写入hostPath卷

### hostPath示例

```bash
[root@k8s-master01 emptydir]# cd ..
[root@k8s-master01 volume]# mkdir hostpath
[root@k8s-master01 volume]# cd hostpath/
[root@k8s-master01 hostpath]# vim pod-hostpath.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-hostpath
  namespace: default
spec:
  containers:
  - name: myapp-pod
    image: registry.cn-beijing.aliyuncs.com/google_registry/myapp:v1
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: hostpath-dir-volume
      mountPath: /test-k8s/hostpath-dir
    - name: hostpath-file-volume
      mountPath: /test/hostpath-file/test.conf
  volumes:
  - name: hostpath-dir-volume
    hostPath:
      # 宿主机目录
      path: /k8s/hostpath-dir
      # hostPath 卷指定 type,如果目录不存在则创建(可创建多层目录)
      type: DirectoryOrCreate
  - name: hostpath-file-volume
    hostPath:
      path: /k8s2/hostpath-file/test.conf
      # 如果文件不存在则创建. 前提: 文件所在目录必须存在  目录不存在则不能创建文件
      type: FileOrCreate
# 创建Pod并查看状态
[root@k8s-master01 hostpath]# kubectl apply -f pod-hostpath.yaml 
pod/pod-hostpath created
[root@k8s-master01 hostpath]# kubectl describe pod pod-hostpath | grep -i event -A 10
Events:
  Type     Reason       Age                From               Message
  ----     ------       ----               ----               -------
  Normal   Scheduled    44s                default-scheduler  Successfully assigned default/pod-hostpath to k8s-master03
  Warning  FailedMount  13s (x7 over 44s)  kubelet            MountVolume.SetUp failed for volume "hostpath-file-volume" : open /k8s2/hostpath-file/test.conf: no such file or directory
# 可以看到由于k8s-master03上未创建/k8s2/hostpath-file目录 pod创建失败了, 一直是ContainerCreating状态
# 在k8s-master03主机上创建目录/k8s2/hostpath-file
[root@k8s-master01 hostpath]# ssh k8s-master03
Last login: Tue Mar 15 23:52:01 2022 from k8s-master01
[root@k8s-master03 ~]# mkdir -p /k8s2/hostpath-file
[root@k8s-master03 ~]# exit
logout
Connection to k8s-master03 closed.
# 创建后再应用一下yaml文件
[root@k8s-master01 hostpath]# kubectl apply -f pod-hostpath.yaml 
pod/pod-hostpath configured
[root@k8s-master01 hostpath]# kubectl get po pod-hostpath -owide
NAME           READY   STATUS    RESTARTS   AGE     IP              NODE           NOMINATED NODE   READINESS GATES
pod-hostpath   1/1     Running   0          5m24s   172.18.195.33   k8s-master03   <none>           <none>
[root@k8s-master01 hostpath]# kubectl describe po pod-hostpath 
Name:         pod-hostpath
Namespace:    default
Priority:     0
Node:         k8s-master03/192.168.43.185
Start Time:   Wed, 16 Mar 2022 17:29:40 +0800
Labels:       <none>
Annotations:  cni.projectcalico.org/containerID: b387d7830097b150b51e584721dc538d2e088eb89d2d6ddbeede8446416cb378
              cni.projectcalico.org/podIP: 172.18.195.33/32
              cni.projectcalico.org/podIPs: 172.18.195.33/32
Status:       Running
IP:           172.18.195.33
IPs:
  IP:  172.18.195.33
Containers:
  myapp-pod:
    Container ID:   containerd://00911225423dbcc01b05b47e7fbef3f64b05de72182d8da9766c46650f54ae44
    Image:          registry.cn-beijing.aliyuncs.com/google_registry/myapp:v1
    Image ID:       registry.cn-beijing.aliyuncs.com/google_registry/myapp@sha256:9eeca44ba2d410e54fccc54cbe9c021802aa8b9836a0bcf3d3229354e4c8870e
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Wed, 16 Mar 2022 17:33:50 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /test-k8s/hostpath-dir from hostpath-dir-volume (rw)
      /test/hostpath-file/test.conf from hostpath-file-volume (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-jlhk2 (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  hostpath-dir-volume:
    Type:          HostPath (bare host directory volume)
    Path:          /k8s/hostpath-dir
    HostPathType:  DirectoryOrCreate
  hostpath-file-volume:
    Type:          HostPath (bare host directory volume)
    Path:          /k8s2/hostpath-file/test.conf
    HostPathType:  FileOrCreate
  kube-api-access-jlhk2:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason       Age                 From               Message
  ----     ------       ----                ----               -------
  Normal   Scheduled    6m                  default-scheduler  Successfully assigned default/pod-hostpath to k8s-master03
  Warning  FailedMount  3m57s               kubelet            Unable to attach or mount volumes: unmounted volumes=[hostpath-file-volume], unattached volumes=[hostpath-file-volume kube-api-access-jlhk2 hostpath-dir-volume]: timed out waiting for the condition
  Warning  FailedMount  3m52s (x9 over 6m)  kubelet            MountVolume.SetUp failed for volume "hostpath-file-volume" : open /k8s2/hostpath-file/test.conf: no such file or directory
  Normal   Pulled       110s                kubelet            Container image "registry.cn-beijing.aliyuncs.com/google_registry/myapp:v1" already present on machine
  Normal   Created      110s                kubelet            Created container myapp-pod
  Normal   Started      110s                kubelet            Started container myapp-pod
# pod被调度到了k8s-master03上并处于运行状态
```

### 验证hostPath

#### 宿主机中操作

切换到k8s-master03节点

```bash
# 查看宿主机空目录和空文件
[root@k8s-master01 hostpath]# ssh k8s-master03
Last login: Wed Mar 16 17:31:58 2022 from k8s-master01
[root@k8s-master03 ~]# ll /k8s/hostpath-dir/
total 0
[root@k8s-master03 ~]# ll /k8s2/hostpath-file/test.conf 
-rw-r--r-- 1 root root 0 Mar 16 17:33 /k8s2/hostpath-file/test.conf
# 对空目录进行操作
[root@k8s-master03 ~]# touch /k8s/hostpath-dir/file1.txt
# 对挂载的文件操作
[root@k8s-master03 ~]# echo "Message from host: k8s-master03" >> /k8s2/hostpath-file/test.conf 
```

#### 容器内操作

切换到k8s-master01主节点

```bash
[root@k8s-master03 ~]# exit
logout
Connection to k8s-master03 closed.
# 进入容器内操作
[root@k8s-master01 hostpath]# kubectl exec -it pod-hostpath -c myapp-pod -- sh
/ # cd /test-k8s/hostpath-dir
/test-k8s/hostpath-dir # ls
file1.txt
/test-k8s/hostpath-dir # echo "Message from container: myapp-pod" >> file1.txt 
/test-k8s/hostpath-dir # cd /test/hostpath-file/
/test/hostpath-file # ls
test.conf
/test/hostpath-file # cat test.conf 
Message from host: k8s-master03
/test/hostpath-file # echo "Message from container: myapp-pod" >> test.conf 
/test/hostpath-file # cat test.conf 
Message from host: k8s-master03
Message from container: myapp-pod

# 切换到主机k8s-master03查看数据卷同步情况
[root@k8s-master01 hostpath]# ssh k8s-master03
Last login: Wed Mar 16 17:39:57 2022 from k8s-master01
[root@k8s-master03 ~]# cat /k8s/hostpath-dir/file1.txt 
Message from container: myapp-pod
[root@k8s-master03 ~]# cat /k8s2/hostpath-file/test.conf 
Message from host: k8s-master03
Message from container: myapp-pod
```

## 挂载NFS

需要结合NFS博客实践：[Kubernetes存储之搭建NFS](https://www.deemoprobe.com/yunv/k-nfs/)，博客中k8s-node02作为nfs服务的客户端。

```bash
[root@k8s-master01 hostpath]# cd ..
[root@k8s-master01 volume]# mkdir nfs;cd nfs
# 先给k8s-node02打上nfs=bingo的标签
[root@k8s-master01 ~]# kubectl label nodes k8s-node02 nfs=bingo
node/k8s-node02 labeled
[root@k8s-master01 nfs]# vim pod-nfs.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: pod-nfs
  namespace: default
spec:
  containers:
  - name: myapp-pod
    image: registry.cn-beijing.aliyuncs.com/google_registry/myapp:v1
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - mountPath: /nfs/data # 容器挂载目录
      name: nfs-volume
  nodeSelector:
    nfs: bingo # 调度到nfs客户端
  volumes:
  - name: nfs-volume
    nfs: # nfs主机和目录
      server: 192.168.43.212
      path: /data
[root@k8s-master01 nfs]# kubectl apply -f pod-nfs.yaml 
pod/pod-nfs created
[root@k8s-master01 nfs]# kubectl get po pod-nfs -owide
NAME      READY   STATUS    RESTARTS   AGE     IP              NODE         NOMINATED NODE   READINESS GATES
pod-nfs   1/1     Running   0          3m25s   172.27.14.220   k8s-node02   <none>           <none>
# 进入容器查看，可见NFS搭建博客中的文件均已共享到/nfs/data目录，这些文件是在Kubernetes存储之搭建NFS博客中创建的
[root@k8s-master01 nfs]# kubectl exec -it pod-nfs -c myapp-pod -- sh
/ # cd /nfs/data/
/nfs/data # ls
client      client.log  server      server.log
/nfs/data # cat client.log 
Message from nfs-client: k8s-node02
/nfs/data # cat server.log 
Message from nfs-server: nfs-server
# 添加容器访问记录
/nfs/data # echo "Message from container: myapp-pod in k8s-node02" >> client.log 

# 去nfs服务端查看
[root@nfs-server ~]# cd /data/
[root@nfs-server data]# ls
client  client.log  server  server.log
[root@nfs-server data]# cat client.log 
Message from nfs-client: k8s-node02
Message from container: myapp-pod in k8s-node02
```
