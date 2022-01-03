# Kubernetes基础之Pod控制器

## 控制器

Kubernetes 中内建了很多 controller(控制器),用来确保pod资源符合预期的状态,控制 Pod 的状态和行为.

## 控制器类型

- ReplicationController(rc)
- ReplicaSet(rs)
- Deployment(deploy)
- DaemonSet(ds)
- StatefulSet(sts)
- Job/CronJob(cj)
- Horizontal Pod Autoscaling(hpa)

```shell
# 可以使用kubectl explain命令查看k8s API资源对象描述信息
[root@k8s-master ~]# kubectl explain rs

# 查看API资源列表
[root@k8s-master ~]# kubectl api-resources
```

### ReplicaSet 和 ReplicationController

ReplicationController(RC)用来确保容器应用的副本数始终保持在用户定义的副本数,即如果有容器异常退出,会自动创建新的 Pod 来替代;而如果异常多出来的容器也会自动回收.

在新版本的Kubernetes中建议使用ReplicaSet来取代ReplicationController,ReplicaSet规则跟ReplicationController没有本质的不同,唯一区别是ReplicaSet支持`selector`选择器.

虽然ReplicaSets可以独立使用,但如今它主要被Deployment等更高一级的资源用作协调Pod的创建、删除和更新的机制.

#### rc实现pod动态缩放

- 当前RC和pod情况是

```shell
[root@k8s-master ~]# kubectl get pods,rc
NAME                         READY   STATUS    RESTARTS   AGE
pod/mysql-hdg66              1/1     Running   3          24h
pod/myweb-ctzhn              1/1     Running   3          24h
pod/myweb-dm94j              1/1     Running   3          24h

NAME                          DESIRED   CURRENT   READY   AGE
replicationcontroller/mysql   1         1         1       24h
replicationcontroller/myweb   2         2         2       24h
```

- 增加pod-mysql的副本(RC)数

```shell
[root@k8s-master ~]# kubectl scale rc mysql --replicas=3
replicationcontroller/mysql scaled
```

- 减少pod-mysql的副本(RC)数

```shell
[root@k8s-master ~]# kubectl scale rc mysql --replicas=1
replicationcontroller/mysql scaled
```

### 2.2. Deployment

Deployment 是一种更高级别的 API 资源对象,为 Pods 和 ReplicaSets 提供声明式的更新能力.它以类似于 `kubectl rolling-update` 的方式更新其底层 ReplicaSet 及其 Pod. 如果需要这种滚动更新功能,推荐使用 Deployment.

Deployments 的典型用例:

- 创建 Deployment 将 ReplicaSet 上线.ReplicaSet 在后台创建 Pods.检查 ReplicaSet 的上线状态,查看其是否成功.
- 通过更新 Deployment 的 PodTemplateSpec,声明 Pod 的新状态.新的 ReplicaSet 会被创建,Deployment 将 Pod 从旧 ReplicaSet 迁移到新 ReplicaSet. 每个新的 ReplicaSet 都会更新 Deployment 的修订版本.
- 如果 Deployment 的当前状态不稳定,回滚到较早的 Deployment 版本.每次回滚都会更新 Deployment 的修订版本.
- 扩大 Deployment 规模以承担更多负载.
- 暂停 Deployment 以应用对 PodTemplateSpec 所作的多项修改,然后恢复其执行以启动新的上线版本.
- 使用 Deployment 状态 来判定上线过程是否出现停滞.
- 清理较旧的不再需要的 ReplicaSet.

#### 创建 Deployment

下面是 Deployment 示例.其中创建了一个 ReplicaSet,负责启动三个 nginx Pods:

```shell
# 创建yaml文件
[root@k8s-master k_base]# vi nginx-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.16.1
        ports:
        - containerPort: 80
[root@k8s-master k_base]# kubectl apply -f nginx-deploy.yaml
deployment.apps/nginx-deploy created
[root@k8s-master k_base]# kubectl get po,deploy,rs
NAME                                READY   STATUS    RESTARTS   AGE
pod/nginx-deploy-559d658b74-27jn4   1/1     Running   0          100s
pod/nginx-deploy-559d658b74-hzdr2   1/1     Running   0          100s
pod/nginx-deploy-559d658b74-v7rhq   1/1     Running   0          100s

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deploy   3/3     3            3           101s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-deploy-559d658b74   3         3         3       100s
# 查看标签
[root@k8s-master k_base]# kubectl get po --show-labels
NAME                            READY   STATUS    RESTARTS   AGE     LABELS
nginx-deploy-559d658b74-27jn4   1/1     Running   0          2m20s   app=nginx,pod-template-hash=559d658b74
nginx-deploy-559d658b74-hzdr2   1/1     Running   0          2m20s   app=nginx,pod-template-hash=559d658b74
nginx-deploy-559d658b74-v7rhq   1/1     Running   0          2m20s   app=nginx,pod-template-hash=559d658b74
# 扩缩容
[root@k8s-master k_base]# kubectl scale deployment nginx-deploy --replicas=2
deployment.apps/nginx-deploy scaled
[root@k8s-master k_base]# kubectl get po,deploy,rs
NAME                                READY   STATUS    RESTARTS   AGE
pod/nginx-deploy-559d658b74-27jn4   1/1     Running   0          10m
pod/nginx-deploy-559d658b74-hzdr2   1/1     Running   0          10m

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deploy   2/2     2            2           10m

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-deploy-559d658b74   2         2         2       10m
# 查看pod详情
[root@k8s-master k_base]# kubectl get pods -o wide
NAME                            READY   STATUS    RESTARTS   AGE   IP              NODE        NOMINATED NODE   READINESS GATES
nginx-deploy-559d658b74-27jn4   1/1     Running   0          11m   172.16.36.105   k8s-node1   <none>           <none>
nginx-deploy-559d658b74-hzdr2   1/1     Running   0          11m   172.16.36.106   k8s-node1   <none>           <none>
# 访问nginx
[root@k8s-master k_base]# curl 172.16.36.105
...
<title>Welcome to nginx!</title>
...
```

说明:

- ReplicaSet 的名称始终被格式化为[DEPLOYMENT-NAME]-[RANDOM-STRING].`RANDOM-STRING`随机生成,并使用 pod-template-hash 作为选择器和标签.

- Deployment 控制器将 pod-template-hash 标签添加到 Deployment 创建或使用的每个 ReplicaSet .此标签可确保 Deployment 的子 ReplicaSets 不重叠.因此不可修改.

- 注意Deployment、ReplicaSet和Pod三者的名称关系

```shell
[root@k8s-master k_base]# kubectl get deploy,rs -o wide
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES         SELECTOR
deployment.apps/nginx-deploy   3/3     3            3           45m   nginx        nginx:1.18.0   app=nginx

NAME                                      DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES         SELECTOR
replicaset.apps/nginx-deploy-67dfd6c8f9   3         3         3       16m   nginx        nginx:1.18.0   app=nginx,pod-template-hash=67dfd6c8f9
```

#### 更新 Deployment

Deployment 可确保在更新时仅关闭一定数量的 Pods.默认情况下,它确保至少 75% 所需 Pods 在运行(25%为容忍的最大不可用量).更新时不会先删除旧的pod,而是先新建一个pod.新pod运行时,才会删除对应老的pod.一切的前提都是为了满足上述的条件.

备注: 如果需要更新Deployment,最好通过yaml文件更新,这样回滚到任何版本都非常便捷,而且更容易追述.

```shell
# 方式一: 直接修改镜像[不推荐]
# 执行下面命令后修改对于镜像版本即可, 该方法不会记录命令,通过kubectl rollout history deployment/nginx-deployment 无法查询
[root@k8s-master k_base]# kubectl edit deploy/nginx-deploy

# 方式二: 命令行更新image[可使用]
[root@k8s-master k_base]# kubectl set image deploy/nginx-deploy nginx=nginx:1.18.0 --record
deployment.apps/nginx-deploy image updated
[root@k8s-master k_base]# kubectl get po,deploy,rs
NAME                                READY   STATUS    RESTARTS   AGE
pod/nginx-deploy-67dfd6c8f9-gxj29   1/1     Running   0          43s
pod/nginx-deploy-67dfd6c8f9-xm68b   1/1     Running   0          45s

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deploy   2/2     2            2           17m

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-deploy-559d658b74   0         0         0       17m
replicaset.apps/nginx-deploy-67dfd6c8f9   2         2         2       45s
[root@k8s-master k_base]# kubectl get pods -o wide
NAME                            READY   STATUS    RESTARTS   AGE     IP              NODE        NOMINATED NODE   READINESS GATES
nginx-deploy-67dfd6c8f9-gxj29   1/1     Running   0          3m35s   172.16.36.109   k8s-node1   <none>           <none>
nginx-deploy-67dfd6c8f9-xm68b   1/1     Running   0          3m37s   172.16.36.108   k8s-node1   <none>           <none>
# 方式三: 使用yaml文件更新版本
# 当前nginx 版本是1.16.1, 更新到1.18.0
[root@k8s-master k_base]# kubectl get deploy -o wide --show-labels
NAME           READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES         SELECTOR    LABELS
nginx-deploy   3/3     3            3           63m   nginx        nginx:1.16.1   app=nginx   app=nginx
[root@k8s-master k_base]# ll
total 8
-rw-r--r-- 1 root root 337 Dec 21 09:59 nginx-deploy-1161.yaml
-rw-r--r-- 1 root root 337 Dec 21 11:01 nginx-deploy-1180.yaml
[root@k8s-master k_base]# cat nginx-deploy-1161.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.16.1
        ports:
        - containerPort: 80
[root@k8s-master k_base]# cat nginx-deploy-1180.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.18.0
        ports:
        - containerPort: 80
# 应用镜像为1180的同样的配置文件
# --record 参数可以记录命令,通过 kubectl rollout history deployment/nginx-deployment 可查询
[root@k8s-master k_base]# kubectl apply -f nginx-deploy-1180.yaml --record
deployment.apps/nginx-deploy configured
# 如果正在更新, 可看到下面日志
[root@k8s-master k_base]# kubectl rollout status deploy/nginx-deploy
Waiting for deployment "nginx-deploy" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deploy" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deploy" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deploy" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deploy" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deploy" rollout to finish: 2 old replicas are pending termination...
Waiting for deployment "nginx-deploy" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx-deploy" rollout to finish: 1 old replicas are pending termination...
deployment "nginx-deploy" successfully rolled out
# 更新结束后, 可看到下面日志
[root@k8s-master k_base]# kubectl rollout status deploy/nginx-deploy
deployment "nginx-deploy" successfully rolled out
```

#### 回滚 Deployment

```shell
# 回滚到上一个版本
[root@k8s-master k_base]# kubectl rollout undo deployment/nginx-deploy
deployment.apps/nginx-deploy rolled back
[root@k8s-master k_base]# kubectl get po,deploy,rs
NAME                                READY   STATUS        RESTARTS   AGE
pod/nginx-deploy-559d658b74-9dgg8   1/1     Running       0          8s
pod/nginx-deploy-559d658b74-pgrqv   1/1     Running       0          10s
pod/nginx-deploy-67dfd6c8f9-gxj29   0/1     Terminating   0          4m42s
pod/nginx-deploy-67dfd6c8f9-xm68b   0/1     Terminating   0          4m44s

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deploy   2/2     2            2           21m

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-deploy-559d658b74   2         2         2       21m
replicaset.apps/nginx-deploy-67dfd6c8f9   0         0         0       4m44s
[root@k8s-master k_base]# kubectl get pods -o wide
NAME                            READY   STATUS    RESTARTS   AGE   IP              NODE        NOMINATED NODE   READINESS GATES
nginx-deploy-559d658b74-9dgg8   1/1     Running   0          59s   172.16.36.111   k8s-node1   <none>           <none>
nginx-deploy-559d658b74-pgrqv   1/1     Running   0          61s   172.16.36.110   k8s-node1   <none>           <none>

# 直接用Yaml文件回退[更新]为对应镜像, 比如 1180-->1161
[root@k8s-master k_base]# kubectl apply -f nginx-deploy-1161.yaml --record
```

#### 回退到历史版本

```shell
# 查看历史版本
[root@k8s-master k_base]# kubectl rollout history deploy/nginx-deploy
deployment.apps/nginx-deploy 
REVISION  CHANGE-CAUSE
5         kubectl set image deploy/nginx-deploy nginx=nginx:1.18.0 --record=true
6         kubectl apply --filename=nginx-deploy-1180.yaml --record=true
# 查看历史版本信息
[root@k8s-master k_base]# kubectl rollout history deploy/nginx-deploy --revision=6
deployment.apps/nginx-deploy with revision #6
Pod Template:
  Labels:       app=nginx
        pod-template-hash=67dfd6c8f9
  Annotations:  kubernetes.io/change-cause: kubectl apply --filename=nginx-deploy-1180.yaml --record=true
  Containers:
   nginx:
    Image:      nginx:1.18.0
    Port:       80/TCP
    Host Port:  0/TCP
    Environment:        <none>
    Mounts:     <none>
  Volumes:      <none>
# 可以使用 --revision参数指定回退到某个历史版本
kubectl rollout undo deployment/nginx-deploy --to-revision=6
# 暂停 deployment 的更新
kubectl rollout pause deployment/nginx-deploy
```

#### 查看更新详情

```shell
[root@k8s-master k_base]# kubectl describe po nginx-deploy-559d658b74-9dgg8
Labels:       app=nginx
              pod-template-hash=559d658b74
...
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  111s  default-scheduler  Successfully assigned default/nginx-deploy-559d658b74-9dgg8 to k8s-node1
  Normal  Pulled     110s  kubelet            Container image "nginx:1.16.1" already present on machine
  Normal  Created    110s  kubelet            Created container nginx
  Normal  Started    109s  kubelet            Started container nginx
[root@k8s-master k_base]# kubectl set image deployment/nginx-deploy nginx=nginx:1.18.0 --record
deployment.apps/nginx-deploy image updated
[root@k8s-master k_base]# kubectl get po,deploy,rs
NAME                                READY   STATUS    RESTARTS   AGE
pod/nginx-deploy-67dfd6c8f9-mkmmn   1/1     Running   0          70s
pod/nginx-deploy-67dfd6c8f9-tj2j8   1/1     Running   0          72s

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deploy   2/2     2            2           26m

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-deploy-559d658b74   0         0         0       26m
replicaset.apps/nginx-deploy-67dfd6c8f9   2         2         2       9m17s
[root@k8s-master k_base]# kubectl describe po nginx-deploy-67dfd6c8f9-mkmmn
Name:         nginx-deploy-67dfd6c8f9-mkmmn
Namespace:    default
...
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  95s   default-scheduler  Successfully assigned default/nginx-deploy-67dfd6c8f9-mkmmn to k8s-node1
  Normal  Pulled     93s   kubelet            Container image "nginx:1.18.0" already present on machine
  Normal  Created    93s   kubelet            Created container nginx
  Normal  Started    93s   kubelet            Started container nginx
# 查看回滚状态
[root@k8s-master k_base]# kubectl rollout status deploy nginx-deploy
deployment "nginx-deploy" successfully rolled out
# 历史记录
[root@k8s-master k_base]# kubectl rollout history deploy nginx-deploy
deployment.apps/nginx-deploy
REVISION  CHANGE-CAUSE
3         <none>
4         kubectl set image deployment/nginx-deploy nginx=nginx:1.18.0 --record=true
# 查看deployment详情
[root@k8s-master k_base]# kubectl describe deploy nginx-deploy
Name:                   nginx-deploy
Namespace:              default
CreationTimestamp:      Mon, 23 Nov 2020 02:24:33 -0500
Labels:                 app=nginx
Annotations:            deployment.kubernetes.io/revision: 4
                        kubernetes.io/change-cause: kubectl set image deployment/nginx-deploy nginx=nginx:1.18.0 --record=true
Selector:               app=nginx
Replicas:               2 desired | 2 updated | 2 total | 2 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:        nginx:1.18.0
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   nginx-deploy-67dfd6c8f9 (2/2 replicas created)
Events:
  Type    Reason             Age                From                   Message
  ----    ------             ----               ----                   -------
  Normal  ScalingReplicaSet  40m                deployment-controller  Scaled up replica set nginx-deploy-559d658b74 to 3
  Normal  ScalingReplicaSet  30m                deployment-controller  Scaled down replica set nginx-deploy-559d658b74 to 2
  Normal  ScalingReplicaSet  18m                deployment-controller  Scaled up replica set nginx-deploy-559d658b74 to 1
  Normal  ScalingReplicaSet  18m                deployment-controller  Scaled up replica set nginx-deploy-559d658b74 to 2
  Normal  ScalingReplicaSet  18m                deployment-controller  Scaled down replica set nginx-deploy-67dfd6c8f9 to 1
  Normal  ScalingReplicaSet  18m                deployment-controller  Scaled down replica set nginx-deploy-67dfd6c8f9 to 0
  Normal  ScalingReplicaSet  15m (x2 over 23m)  deployment-controller  Scaled up replica set nginx-deploy-67dfd6c8f9 to 1
  Normal  ScalingReplicaSet  15m (x2 over 23m)  deployment-controller  Scaled up replica set nginx-deploy-67dfd6c8f9 to 2
  Normal  ScalingReplicaSet  15m (x2 over 23m)  deployment-controller  Scaled down replica set nginx-deploy-559d658b74 to 1
  Normal  ScalingReplicaSet  15m (x2 over 23m)  deployment-controller  Scaled down replica set nginx-deploy-559d658b74 to 0

# 其他操作
# 1. 查看deploy详情
[root@k8s-master k_base]# kubectl get deploy nginx-deploy -o wide --show-labels
# 2. 查看rs详情
[root@k8s-master k_base]# kubectl get rs -o wide --show-labels
# 3. 查看po详情
[root@k8s-master k_base]# kubectl get pod -o wide --show-labels
# 扩缩容
[root@k8s-master k_base]# kubectl scale deploy/nginx-deploy --replicas=5
# 可以在 Deployment 中设置 .spec.revisionHistoryLimit,以指定保留多少该 Deployment 的 ReplicaSets数量
```

### DaemonSet

DaemonSet 确保全部(或者一些)Node节点上运行一个 Pod 的副本.当有 Node 加入集群时,也会为它们新增一个 Pod,当有 Node 从集群移除时,这些 Pod 也会被回收.删除 DaemonSet 将会删除它创建的所有 Pod.

DaemonSet典型用法:

- 在每个节点上运行集群存储 DaemonSet,例如 glusterd、ceph.
- 在每个节点上运行日志收集 DaemonSet,例如 fluentd、logstash.
- 在每个节点上运行监控 DaemonSet,例如 Prometheus Node Exporter、Flowmill、Sysdig 代理、collectd、Dynatrace OneAgent、AppDynamics 代理、Datadog 代理、New Relic 代理、Ganglia gmond 或者 Instana 代理.

一个简单的用法是在所有的节点上都启动一个 DaemonSet,并作为每种类型的 daemon 使用.

一个稍微复杂的用法是单独对每种 daemon 类型使用一种DaemonSet.这样有多个 DaemonSet,但具有不同的标识,并且对不同硬件类型具有不同的内存、CPU 要求.

```shell
[root@k8s-master k_base]# kubectl explain ds
KIND:     DaemonSet
VERSION:  apps/v1

DESCRIPTION:
     DaemonSet represents the configuration of a daemon set.

FIELDS:
...
# 创建yaml
[root@k8s-master k_base]# pwd
/app/kubernetes/k_base
[root@k8s-master k_base]# cat daemonset.yaml 
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: default
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      # 允许在master节点运行
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: registry.cn-beijing.aliyuncs.com/google_registry/fluentd:v2.5.2
        resources:
          limits:
            cpu: 1
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      # 优雅关闭应用,时间设置.超过该时间会强制关闭,默认30秒
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
[root@k8s-master k_base]# kubectl apply -f daemonset.yaml 
daemonset.apps/fluentd-elasticsearch created
[root@k8s-master k_base]# kubectl get ds -o wide
NAME                    DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE    CONTAINERS              IMAGES                                                            SELECTOR
fluentd-elasticsearch   2         2         2       2            2           <none>          103s   fluentd-elasticsearch   registry.cn-beijing.aliyuncs.com/google_registry/fluentd:v2.5.2   name=fluentd-elasticsearch
# 我这里只启了master和node1节点, 可见DaemonSet在每个节点上都有相应的pod
[root@k8s-master k_base]# kubectl get po -o wide
NAME                            READY   STATUS             RESTARTS   AGE    IP               NODE         NOMINATED NODE   READINESS GATES
fluentd-elasticsearch-cbl4x     1/1     Running            0          109s   172.16.36.106    k8s-node1    <none>           <none>
fluentd-elasticsearch-pzlpv     1/1     Running            0          108s   172.16.235.250   k8s-master   <none>           <none>
```

### Job

Job 负责批处理任务,即仅执行一次的任务,它保证批处理任务的一个或多个 Pod 成功结束.

```shell
[root@k8s-master k_base]# kubectl explain job
KIND:     Job
VERSION:  batch/v1

DESCRIPTION:
     Job represents the configuration of a single job.

FIELDS:
...
# 创建yaml
[root@k8s-master k_base]# vi job-example.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    metadata:
      name: pi
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(1000)"]
      restartPolicy: Neve
[root@k8s-master k_base]# kubectl get po
NAME                      READY   STATUS      RESTARTS   AGE
daemonset-example-f57kg   1/1     Running     0          13m
pi-2nlj5                  0/1     Completed   0          4m
[root@k8s-master k_base]# kubectl get job
NAME   COMPLETIONS   DURATION   AGE
pi     1/1           3m19s      4m15s
[root@k8s-master k_base]# kubectl get po pi-2nlj5 -o wide
NAME       READY   STATUS      RESTARTS   AGE     IP              NODE        NOMINATED NODE   READINESS GATES
pi-2nlj5   0/1     Completed   0          4m54s   172.16.36.119   k8s-node1   <none>           <none>
# 查看日志可以看到job执行的结果
# 计算出了圆周率后1000位
```

![20201123180138](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20201123180138.png)

### CronJob

Cron Job 管理基于时间的 Job,即:

在给定时间点只运行一次
周期性地在给定时间点运行

典型的用法示例:

在给你写的时间点调度 Job 运行
创建周期性运行的 Job,例如: 数据库备份、发送邮件

```shell
[root@k8s-master k_base]# kubectl explain cj
KIND:     CronJob
VERSION:  batch/v1beta1

DESCRIPTION:
     CronJob represents the configuration of a single cron job.

FIELDS:
...
# 创建cronjob yaml文件
[root@k8s-master k_base]# vi cronjob-example.yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello CronJob
          restartPolicy:  OnFailure
[root@k8s-master k_base]# kubectl apply -f cronjob-example.yaml
cronjob.batch/hello created
[root@k8s-master k_base]# kubectl get cj
NAME    SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
hello   */1 * * * *   False     0        44s             69s
[root@k8s-master k_base]# kubectl get po
NAME                     READY   STATUS      RESTARTS   AGE
hello-1606186260-4pbtt   0/1     Completed   0          100s
hello-1606186320-gxtkn   0/1     Completed   0          39s
# 查看输出日志
[root@k8s-master k_base]# kubectl logs hello-1606186260-4pbtt
Tue Nov 24 02:51:25 UTC 2020
Hello CronJob
[root@k8s-master k_base]# kubectl logs hello-1606186320-gxtkn
Tue Nov 24 02:52:26 UTC 2020
Hello CronJob
[root@k8s-master k_base]# kubectl get job
NAME               COMPLETIONS   DURATION   AGE
hello-1606186260   1/1           18s        2m42s
hello-1606186320   1/1           18s        101s
hello-1606186380   1/1           30s        41s
# 删除cronjob
[root@k8s-master k_base]# kubectl delete cronjob hello
cronjob.batch "hello" deleted
[root@k8s-master k_base]# kubectl get job
No resources found in default namespace.
```

### StatefulSet

StatefulSet 是用来管理有状态应用的工作负载 API 对象.

StatefulSet 中的 Pod 拥有独一无二的身份标识.这个标识基于 StatefulSet 控制器分配给每个 Pod 的唯一顺序索引.Pod 的名称的形式为`<statefulset name>-<ordinal index>` .例如: web的StatefulSet 拥有两个副本,所以它创建了两个 Pod: web-0和web-1.

和 Deployment 相同的是,StatefulSet 管理了基于相同容器定义的一组 Pod.但和 Deployment 不同的是,StatefulSet 为它们的每个 Pod 维护了一个固定的 ID.这些 Pod 是基于相同的声明来创建的,但是不能相互替换: 无论怎么调度,每个 Pod 都有一个永久不变的 ID.

使用场景:

- 稳定的、唯一的网络标识符,即Pod重新调度后其PodName和HostName不变[当然IP是会变的]
- 稳定的、持久的存储,即Pod重新调度后还是能访问到相同的持久化数据,基于PVC实现
- 有序的、优雅的部署和缩放
- 有序的、自动的滚动更新

如上面,稳定意味着 Pod 调度或重调度的整个过程是有持久性的.

如果应用程序不需要任何稳定的标识符或有序的部署、删除或伸缩,则应该使用由一组无状态的副本控制器提供的工作负载来部署应用程序,比如使用 Deployment 或者 ReplicaSet 可能更适用于无状态应用部署需要.

#### 限制

- 给定 Pod 的存储必须由 PersistentVolume(PV) 驱动基于所请求的 storage class 来提供,或者由管理员预先提供.
- 删除或者收缩 StatefulSet 并不会删除它关联的存储卷.这样做是为了保证数据安全,它通常比自动清除 StatefulSet 所有相关的资源更有价值.
- StatefulSet 当前需要 headless 服务来负责 Pod 的网络标识.需要先创建此服务.
- 当删除 StatefulSets 时,StatefulSet 不提供任何终止 Pod 的保证.为了实现 StatefulSet 中的 Pod 可以有序和优雅的终止,可以在删除之前将 StatefulSet 缩放为 0.
- 在默认 Pod 管理策略(OrderedReady) 时使用滚动更新,可能进入需要人工干预才能修复的损坏状态.

#### 有序索引

对于具有 N 个副本的 StatefulSet,StatefulSet 中的每个 Pod 将被分配一个整数序号,从 0 到 N-1,该序号在 StatefulSet 上是唯一的.

StatefulSet 中的每个 Pod 根据 StatefulSet 中的名称和 Pod 的序号来派生出它的主机名.组合主机名的格式为`<statefulset name>-<ordinal index>`.

#### 部署和扩缩保证

- 对于包含 N 个 副本的 StatefulSet,当部署 Pod 时,它们是依次创建的,顺序为 0~(N-1).
- 当删除 Pod 时,它们是逆序终止的,顺序为 (N-1)~0.
- 在将缩放操作应用到 Pod 之前,它前面的所有 Pod 必须是 Running 和 Ready 状态.
- 在 Pod 终止之前,所有的继任者必须完全关闭.

StatefulSet 不应将 pod.Spec.TerminationGracePeriodSeconds 设置为 0.这种做法是不安全的,要强烈阻止.

#### 部署顺序

在下面的 nginx 示例被创建后,会按照 web-0、web-1、web-2 的顺序部署三个 Pod.在 web-0 进入 Running 和 Ready 状态前不会部署 web-1.在 web-1 进入 Running 和 Ready 状态前不会部署 web-2.

如果 web-1 已经处于 Running 和 Ready 状态,而 web-2 尚未部署,在此期间发生了 web-0 运行失败,那么 web-2 将不会被部署,要等到 web-0 部署完成并进入 Running 和 Ready 状态后,才会部署 web-2.

#### 收缩顺序

如果想将示例中的 StatefulSet 收缩为 replicas=1,首先被终止的是 web-2.在 web-2 没有被完全停止和删除前,web-1 不会被终止.当 web-2 已被终止和删除；但web-1 尚未被终止,如果在此期间发生 web-0 运行失败,那么就不会终止 web-1,必须等到 web-0 进入 Running 和 Ready 状态后才会终止 web-1.

```shell
# 查看StatefulSet说明
[root@k8s-master k_base]# kubectl explain sts
KIND:     StatefulSet
VERSION:  apps/v1

DESCRIPTION:
     StatefulSet represents a set of pods with consistent identities. Identities
     are defined as:
     - Network: A single stable DNS and hostname.
     - Storage: As many VolumeClaims as requested. The StatefulSet guarantees
     that a given network identity will always map to the same storage identity.

FIELDS:
...
```

#### StatefulSet实例

```shell
[root@k8s-master k_base]# pwd
/app/kubernetes/k_base
[root@k8s-master k_base]# cat statefulset.yaml 
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: http
  clusterIP: None
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
      app: nginx # has to match .spec.template.metadata.labels
  serviceName: "nginx"
  replicas: 3 # by default is 1
  template:
    metadata:
      labels:
        app: nginx # has to match .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 10   # 默认30秒
      containers:
      - name: nginx
        image: registry.cn-beijing.aliyuncs.com/google_registry/nginx:1.17
        ports:
        - containerPort: 80
          name: http
[root@k8s-master k_base]# kubectl apply -f statefulset.yaml 
service/nginx created
statefulset.apps/web created
[root@k8s-master k_base]# kubectl get svc -o wide
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE    SELECTOR
kubernetes   ClusterIP   192.168.0.1   <none>        443/TCP   4d5h   <none>
nginx        ClusterIP   None          <none>        80/TCP    19s    app=nginx
[root@k8s-master k_base]# kubectl get sts -o wide
NAME   READY   AGE   CONTAINERS   IMAGES
web    3/3     38s   nginx        registry.cn-beijing.aliyuncs.com/google_registry/nginx:1.17
[root@k8s-master k_base]# kubectl get po -o wide
NAME                            READY   STATUS             RESTARTS   AGE     IP               NODE         NOMINATED NODE   READINESS GATES
web-0                           1/1     Running            0          52s     172.16.36.96     k8s-node1    <none>           <none>
web-1                           1/1     Running            0          42s     172.16.36.91     k8s-node1    <none>           <none>
web-2                           1/1     Running            0          39s     172.16.36.101    k8s-node1    <none>           <none>
# 创建busybox Pod进去查看三个nginx Pod的域名信息
[root@k8s-master k_base]# vi busybox.yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - name: busybox
    image: busybox:1.28
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
# 可以看到web-0, web-1, web-2的域名信息, 其他域名信息是我其他nginx服务的信息, 忽略即可
[root@k8s-master k_base]# kubectl exec -it busybox /bin/sh
/ # nslookup nginx.default.svc.cluster.local
Server:    192.168.0.10
Address 1: 192.168.0.10 kube-dns.kube-system.svc.cluster.local

Name:      nginx.default.svc.cluster.local
Address 1: 172.16.36.91 web-1.nginx.default.svc.cluster.local
Address 2: 172.16.235.235 172-16-235-235.nginx.default.svc.cluster.local
Address 3: 172.16.36.98 172-16-36-98.nginx.default.svc.cluster.local
Address 4: 172.16.36.101 web-2.nginx.default.svc.cluster.local
Address 5: 172.16.36.94 172-16-36-94.nginx.default.svc.cluster.local
Address 6: 172.16.36.100 172-16-36-100.nginx.default.svc.cluster.local
Address 7: 172.16.36.96 web-0.nginx.default.svc.cluster.local
```

### Horizontal Pod Autoscaling(HPA)

顾名思义, Pod 水平自动缩放,提高集群的整体资源利用率.

Horizontal Pod Autoscaling可以根据指标自动伸缩一个Replication Controller、Deployment 或者Replica Set中的Pod数量

下面是HPA工作简单模型图:

![20201222152148](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20201222152148.png)

主要分两个版本:

- autoscaling/v1: v1版本只支持 cpu
- autoscaling/v2beta2: v2beta2版本支持 自定义 ,内存 ,但是目前也仅仅是处于beta阶段

#### 指标从哪里来

Horizontal Pod AutoScaler被实现为一个控制循环,周期由控制器管理器的–Horizontal Pod AutoScaler sync period标志(默认值为15秒)控制.

在每个期间,控制器管理器都会根据每个HorizontalPodAutoScaler定义中指定的度量来查询资源利用率.控制器管理器从资源度量api(针对每个pod的资源度量)或自定义度量api(针对所有其他度量)获取度量.

对于每个pod的资源度量(如cpu),控制器从horizontalpodautoscaler针对每个pod的资源度量api获取度量.然后,如果设置了目标利用率值,则控制器将利用率值计算为每个pod中容器上等效资源请求的百分比.如果设置了目标原始值,则直接使用原始度量值.然后,控制器获取所有目标pod的利用率平均值或原始值(取决于指定的目标类型),并生成用于缩放所需副本数量的比率.

#### 为什么目前能使用的指标是CPU

v1的模板可能是大家平时见到最多的也是最简单的,v1版本的HPA只支持一种指标 —— CPU.传统意义上,弹性伸缩最少也会支持CPU与Memory两种指标,为什么在Kubernetes中只放开了CPU呢?其实最早的HPA是计划同时支持这两种指标的,但是实际的开发测试中发现,内存不是一个非常好的弹性伸缩判断条件.因为和CPU不同,很多内存型的应用,并不会因为HPA弹出新的容器而带来内存的快速回收,因为很多应用的内存都要交给语言层面的VM进行管理,也就是内存的回收是由VM的GC来决定的.这就有可能因为GC时间的差异导致HPA在不恰当的时间点震荡,因此在v1的版本中,HPA就只支持了CPU一种指标.

#### HPA与rolling update的区别

目前在kubernetes中,可以通过直接管理复制控制器来执行滚动更新,也可以使用deployment对象来管理底层副本集.HPA只支持后一种方法: HPA绑定到部署对象,设置部署对象的大小,部署负责设置底层副本集的大小.

HPA不能使用复制控制器的直接操作进行滚动更新,即不能将HPA绑定到复制控制器并进行滚动更新(例如,使用Kubectl滚动更新).这不起作用的原因是,当滚动更新创建新的复制控制器时,HPA将不会绑定到新的复制控制器.

#### HPA怎么使用

1.使用kubectl的方式

```shell
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
```

2.使用yaml创建

```shell
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```

#### 示例

下载国内示例镜像

```shell
# 在集群所有节点都需要执行[主要是node节点]
docker pull registry.cn-beijing.aliyuncs.com/google_registry/hpa-example
```

```shell
# yaml
[root@k8s-master monitor]# cat php-apache.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      hpa: php-apache
  replicas: 1
  template:
    metadata:
      labels:
        hpa: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.cn-beijing.aliyuncs.com/google_registry/hpa-example
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m
apiVersion: v1
kind: Service
metadata:
  name: php-apache
  labels:
    hpa: php-apache
spec:
  ports:
  - port: 80
  selector:
    hpa: php-apache
# 启动
[root@k8s-master monitor]# kubectl apply -f php-apache.yaml 
deployment.apps/php-apache created
service/php-apache created
# 查看
[root@k8s-master monitor]# kubectl get svc,deploy,po | grep php
service/php-apache                                               ClusterIP      192.168.160.106   <none>          80/TCP                       108s
deployment.apps/php-apache                                               1/1     1            1           3m31s
pod/php-apache-544bfd4f54-77rb6                                       1/1     Running   0          3m31s
# 创建HPA
# 当pod中CPU使用率达50%就扩容.最小1个,最大10个
[root@k8s-master monitor]# kubectl autoscale deploy php-apache --cpu-percent=50 --min=1 --max=10
horizontalpodautoscaler.autoscaling/php-apache autoscaled
[root@k8s-master monitor]# kubectl get hpa
NAME         REFERENCE               TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   <unknown>/50%   1         10        0          12s
```

压测php-apache

```shell
# 创建压测Pod并进入, 先不要退出
[root@k8s-master monitor]# kubectl run -i --tty load-test --image=busybox /bin/sh
/ # 
# 另起一窗口可查看到pod在运行
[root@k8s-master ~]# kubectl get po | grep load
load-test                                                         1/1     Running   0          89s
# 在load-test继续中操作
/ # while true; do wget -q -O- http://php-apache.dev.svc.cluster.local; done
OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!Session ended., resume using 'kubectl attach load-test -c load-test -i -t' command when the pod is running
# 负载飙升, 副本数增多
[root@k8s-master ~]# kubectl get hpa
NAME         REFERENCE               TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   250%/50%   1         10        5          11m
# 自动扩容
[root@k8s-master ~]# kubectl get po | grep php
php-apache-544bfd4f54-77rb6                                       1/1     Running   0          20m
php-apache-544bfd4f54-hvh96                                       1/1     Running   0          5m10s
php-apache-544bfd4f54-nqrcz                                       1/1     Running   0          4m54s
php-apache-544bfd4f54-qr48z                                       1/1     Running   0          5m10s
php-apache-544bfd4f54-xzpbq                                       1/1     Running   0          5m10s

# 停止压测后负载降下来
[root@k8s-master ~]# kubectl get hpa
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/50%    1         10        5          15m
[root@k8s-master ~]# kubectl get deploy php-apache
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
php-apache   5/5     5            5           16m

# 再过几分钟发现负载降下来并且副本数也降了
[root@k8s-master ~]# kubectl get po | grep php
php-apache-544bfd4f54-77rb6                                       1/1     Running   0          22m
[root@k8s-master ~]# kubectl get deploy php-apache
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
php-apache   1/1     1            1           22m
[root@k8s-master ~]# kubectl get hpa
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/50%    1         10        1          18m
```

> 说明:停止压测，pod数量也不会立即降下来.而是过段时间后才会慢慢降下来.这是为了防止由于网络原因或者间歇性流量突增、突降，导致pod回收太快后面流量上来后Pod数量不够.
