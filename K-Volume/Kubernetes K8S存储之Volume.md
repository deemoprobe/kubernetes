# Kubernetes K8S存储之Volume

在容器中的文件在磁盘上是临时存放的，当容器关闭时这些临时文件也会被一并清除。这给容器中运行的特殊应用程序带来一些问题.

首先，当容器崩溃时，kubelet 将重新启动容器，容器中的文件将会丢失——因为容器会以干净的状态重建.

其次，当在一个 Pod 中同时运行多个容器时，常常需要在这些容器之间共享文件.

Kubernetes 抽象出 Volume 对象来解决这两个问题.

Kubernetes Volume卷具有明确的生命周期——与包裹它的 Pod 相同。 因此，Volume比 Pod 中运行的任何容器的存活期都长，在容器重新启动时数据也会得到保留。 当然，当一个 Pod 不再存在时，Volume也将不再存在。更重要的是，Kubernetes 可以支持许多类型的Volume卷，Pod 也能同时使用任意数量的Volume卷.

使用卷时，Pod 声明中需要提供卷的类型 (.spec.volumes 字段)和卷挂载的位置 (.spec.containers.volumeMounts 字段).

## Volume类型

目前, Kubernetes支持以下类型的Volume, 详情解释请见[官网: Kubernetes卷说明](https://kubernetes.io/zh/docs/concepts/storage/volumes/)

```shell
awsElasticBlockStore
azureDisk
azureFile
cephfs
cinder
configMap
csi
downwardAPI
emptyDir
fc (fibre channel)
flexVolume
flocker
gcePersistentDisk
gitRepo (deprecated)
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

常用的类型主要有: Secret、ConfigMap、emptyDir、hostPath

## emptyDir

当 Pod 指定到某个节点上时，首先创建的是一个 emptyDir 卷，并且只要 Pod 在该节点上运行，卷就一直存在。就像它的名称表示的那样，卷最初是空的。

尽管 Pod 中每个容器挂载 emptyDir 卷的路径可能相同也可能不同，但是这些容器都可以读写 emptyDir 卷中相同的文件。

如果Pod中有多个容器，其中某个容器重启，不会影响emptyDir 卷中的数据。当 Pod 因为某些原因被删除时，emptyDir 卷中的数据也会永久删除。

注意：容器崩溃并不会导致 Pod 被从节点上移除，因此容器崩溃时 emptyDir 卷中的数据是安全的。

### emptyDir用途

- 缓存空间，例如基于磁盘的归并排序
- 为耗时较长的计算任务提供检查点，以便任务能方便地从崩溃前状态恢复执行
- 在 Web 服务器容器服务数据时，保存内容管理器容器获取的文件

### emptyDir示例

```shell
[root@k8s-master volume]# mkdir emptydir;cd emptydir
[root@k8s-master emptydir]# vi pod_emptydir.yaml 
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
[root@k8s-master emptydir]# kubectl get pod pod-emptydir -o wide
NAME           READY   STATUS    RESTARTS   AGE    IP             NODE        NOMINATED NODE   READINESS GATES
pod-emptydir   2/2     Running   0          104s   172.16.36.99   k8s-node1   <none>           <none>
[root@k8s-master emptydir]# kubectl describe pod pod-emptydir
Name:         pod-emptydir
Namespace:    default
Priority:     0
Node:         k8s-node1/192.168.43.20
Start Time:   Tue, 15 Dec 2020 01:25:21 -0500
Labels:       <none>
Annotations:  cni.projectcalico.org/podIP: 172.16.36.99/32
              cni.projectcalico.org/podIPs: 172.16.36.99/32
Status:       Running
IP:           172.16.36.99
IPs:
  IP:  172.16.36.99
Containers:
...
Volumes:
  cache-volume:
    Type:       EmptyDir (a temporary directory that shares a pod lifetime)
    Medium:     
    SizeLimit:  <unset>
  default-token-64lwm:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-64lwm
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  3m2s   default-scheduler  Successfully assigned default/pod-emptydir to k8s-node1
  Normal  Pulled     3m     kubelet            Container image "registry.cn-beijing.aliyuncs.com/google_registry/myapp:v1" already present on machine
  Normal  Created    3m     kubelet            Created container myapp-pod
  Normal  Started    3m     kubelet            Started container myapp-pod
  Normal  Pulling    3m     kubelet            Pulling image "registry.cn-beijing.aliyuncs.com/google_registry/busybox:1.24"
  Normal  Pulled     2m52s  kubelet            Successfully pulled image "registry.cn-beijing.aliyuncs.com/google_registry/busybox:1.24" in 8.320275043s
  Normal  Created    2m52s  kubelet            Created container busybox-pod
  Normal  Started    2m52s  kubelet            Started container busybox-pod
```

创建了`myapp-pod`和`busybox-pod`两个容器

### emptyDir验证

```shell
# 进入myapp-pod容器
[root@k8s-master emptydir]# kubectl exec -it pod-emptydir -c myapp-pod -- sh
/ # cd /cache/
/cache # ls
/cache # echo "myapp-pod" > myapp-pod
/cache # cat myapp-pod 
myapp-pod
# 进入busybox-pod容器
[root@k8s-master emptydir]# kubectl exec -it pod-emptydir -c busybox-pod -- sh
/ # cd /test/cache/
/test/cache # ls
myapp-pod
/test/cache # cat myapp-pod 
myapp-pod
/test/cache # echo busybox-pod >> myapp-pod 
/test/cache # cat myapp-pod 
myapp-pod
busybox-pod
```

由上可见，一个Pod中多个容器可共享同一个emptyDir卷

## hostPath

hostPath 卷能将主机node节点文件系统上的文件或目录挂载到你的 Pod 中。 虽然这不是大多数 Pod 需要的，但是它为一些应用程序提供了强大的逃生舱。

### hostPath用途

- 运行一个需要访问 Docker 引擎内部机制的容器；请使用 hostPath 挂载 /var/lib/docker 路径
- 在容器中运行 cAdvisor 时，以 hostPath 方式挂载 /sys
- 允许 Pod 指定给定的 hostPath 在运行 Pod 之前是否应该存在，是否应该创建以及应该以什么方式存在

### 支持类型

除了必需的 path 属性之外，用户可以选择性地为 hostPath 卷指定 type。支持的 type 值如下：

| Type取值          | 描述                                                                                                                                                             |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
|                   | 空字符串（默认）用于向后兼容，这意味着在安装 hostPath 卷之前不会执行任何检查                                                                                     |
| DirectoryOrCreate | 如果指定的路径不存在，那么将根据需要创建空目录，权限设置为 0755，具有与 Kubelet 相同的组和所有权                                                                 |
| Directory         | 给定的路径必须存在                                                                                                                                               |
| FileOrCreate      | 如果给定路径的文件不存在，那么将在那里根据需要创建空文件，权限设置为 0644，具有与 Kubelet 相同的组和所有权[前提：文件所在目录必须存在；目录不存在则不能创建文件] |
| File              | 给定路径上的文件必须存在                                                                                                                                         |
| Socket            | 在给定路径上必须存在的 UNIX 套接字                                                                                                                               |
| CharDevice        | 在给定路径上必须存在的字符设备                                                                                                                                   |
| BlockDevice       | 在给定路径上必须存在的块设备                                                                                                                                     |

### 注意事项

当使用这种类型的卷时要小心，因为：

- 具有相同配置（例如从 podTemplate 创建）的多个 Pod 会由于节点上文件的不同而在不同节点上有不同的行为
- 当 Kubernetes 按照计划添加资源感知的调度时，这类调度机制将无法考虑由 hostPath 卷使用的资源
- 基础主机上创建的文件或目录只能由 root 用户写入。需要在 特权容器 中以 root 身份运行进程，或者修改主机上的文件权限以便容器能够写入 hostPath 卷

### hostPath示例

```shell
[root@k8s-master hostpath]# cat pod_hostpath.yaml 
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
      # hostPath 卷指定 type，如果目录不存在则创建(可创建多层目录)
      type: DirectoryOrCreate
  - name: hostpath-file-volume
    hostPath:
      path: /k8s2/hostpath-file/test.conf
      # 如果文件不存在则创建。 前提：文件所在目录必须存在  目录不存在则不能创建文件
      type: FileOrCreate
# 创建Pod并查看状态
[root@k8s-master hostpath]# kubectl apply -f pod_hostpath.yaml
pod/pod-hostpath created
[root@k8s-master hostpath]# kubectl describe po pod-hostpath
..
Events:
  Type     Reason       Age                 From               Message
  ----     ------       ----                ----               -------
  Normal   Scheduled    15m                 default-scheduler  Successfully assigned default/pod-hostpath to k8s-node1
...
  Warning  FailedMount  57s (x15 over 15m)  kubelet            MountVolume.SetUp failed for volume "hostpath-file-volume" : open /k8s2/hostpath-file/test.conf: no such file or directory
# 可以看到由于k8s-node1上未创建/k8s2/hostpath-file目录 pod创建失败了, 一直是ContainerCreating状态
# 创建目录后
[root@k8s-master hostpath]# kubectl apply -f pod_hostpath.yaml
[root@k8s-master hostpath]# kubectl describe po pod-hostpath
Name:         pod-hostpath
Namespace:    default
Priority:     0
Node:         k8s-node1/192.168.43.20
Start Time:   Tue, 15 Dec 2020 02:01:28 -0500
Labels:       <none>
Annotations:  cni.projectcalico.org/podIP: 172.16.36.101/32
              cni.projectcalico.org/podIPs: 172.16.36.101/32
Status:       Running
IP:           172.16.36.101
IPs:
  IP:  172.16.36.101
Containers:
  myapp-pod:
    Container ID:   docker://078b6f513bfab96c6572c17429e210979a563da9d6f55656672b493680b3d2e0
    Image:          registry.cn-beijing.aliyuncs.com/google_registry/myapp:v1
    Image ID:       docker-pullable://wangyanglinux/myapp@sha256:9c3dc30b5219788b2b8a4b065f548b922a34479577befb54b03330999d30d513
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Tue, 15 Dec 2020 02:17:52 -0500
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /test-k8s/hostpath-dir from hostpath-dir-volume (rw)
      /test/hostpath-file/test.conf from hostpath-file-volume (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-64lwm (ro)
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
  default-token-64lwm:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-64lwm
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason       Age                   From               Message
  ----     ------       ----                  ----               -------
  Normal   Scheduled    17m                   default-scheduler  Successfully assigned default/pod-hostpath to k8s-node1
  Warning  FailedMount  3m32s (x5 over 14m)   kubelet            Unable to attach or mount volumes: unmounted volumes=[hostpath-file-volume], unattached volumes=[hostpath-dir-volume hostpath-file-volume default-token-64lwm]: timed out waiting for the condition
  Warning  FailedMount  2m40s (x15 over 17m)  kubelet            MountVolume.SetUp failed for volume "hostpath-file-volume" : open /k8s2/hostpath-file/test.conf: no such file or directory
  Warning  FailedMount  76s (x2 over 12m)     kubelet            Unable to attach or mount volumes: unmounted volumes=[hostpath-file-volume], unattached volumes=[hostpath-file-volume default-token-64lwm hostpath-dir-volume]: timed out waiting for the condition
  Normal   Pulled       37s                   kubelet            Container image "registry.cn-beijing.aliyuncs.com/google_registry/myapp:v1" already present on machine
  Normal   Created      37s                   kubelet            Created container myapp-pod
  Normal   Started      36s                   kubelet            Started container myapp-pod
# pod被调度到了k8s-node1上
```

### 验证hostPath

#### 宿主机中操作

pod被调度到了k8s-node1上, 切换到k8s-node1

```shell
# 可以看到已经创建了空的test.conf文件
[root@k8s-node1 hostpath-file]# pwd
/k8s2/hostpath-file
[root@k8s-node1 hostpath-file]# ll
total 0
-rw-r--r-- 1 root root 0 Dec 15 02:17 test.conf
# 对挂载的目录操作
[root@k8s-node1 hostpath-dir]# pwd
/k8s/hostpath-dir
[root@k8s-node1 hostpath-dir]# echo "dir" >> k8s.info
[root@k8s-node1 hostpath-dir]# date >> k8s.info
[root@k8s-node1 hostpath-dir]# cat k8s.info 
dir
Tue Dec 15 02:31:23 EST 2020
# 对挂载的文件操作
[root@k8s-node1 hostpath-file]# pwd
/k8s2/hostpath-file
[root@k8s-node1 hostpath-file]# echo "file" >> test.conf 
[root@k8s-node1 hostpath-file]# date >> test.conf 
[root@k8s-node1 hostpath-file]# cat test.conf 
file
Tue Dec 15 02:33:17 EST 2020
```

#### 容器内操作

切换到k8s-master

```shell
# 进入容器内操作
[root@k8s-master hostpath]# kubectl exec -it pod-hostpath -c myapp-pod -- /bin/sh
##### 对挂载的目录操作
/ # cd /test-k8s/hostpath-dir
/test-k8s/hostpath-dir # ls
k8s.info
/test-k8s/hostpath-dir # cat k8s.info 
dir
Tue Dec 15 02:31:23 EST 2020
/test-k8s/hostpath-dir # date >> k8s.info 
/test-k8s/hostpath-dir # cat k8s.info 
dir
Tue Dec 15 02:31:23 EST 2020
Tue Dec 15 07:36:20 UTC 2020
##### 对挂载的文件操作
/test-k8s/hostpath-dir # cd /test/hostpath-file/
/test/hostpath-file # ls
test.conf
/test/hostpath-file # cat test.conf 
file
Tue Dec 15 02:33:17 EST 2020
/test/hostpath-file # date >> test.conf 
/test/hostpath-file # cat test.conf 
file
Tue Dec 15 02:33:17 EST 2020
Tue Dec 15 07:36:46 UTC 2020

# k8s-node1节点中的对应hostPath中文件情况已同步
[root@k8s-node1 hostpath-file]# pwd
/k8s2/hostpath-file
[root@k8s-node1 hostpath-file]# cat test.conf 
file
Tue Dec 15 02:33:17 EST 2020
Tue Dec 15 07:36:46 UTC 2020
[root@k8s-node1 hostpath-file]# cd /k8s/hostpath-dir/
[root@k8s-node1 hostpath-dir]# cat k8s.info 
dir
Tue Dec 15 02:31:23 EST 2020
Tue Dec 15 07:36:20 UTC 2020
```
