# Kubernetes基础之Pod控制器

Kubernetes 中内建了很多 controller(控制器)，用来确保pod资源符合预期的状态，控制 Pod 的状态和行为。

## 控制器类型

- ReplicationController(rc)
- ReplicaSet(rs)
- Deployment(deploy)
- DaemonSet(ds)
- StatefulSet(sts)
- Job/CronJob(cj)
- Horizontal Pod Autoscaling(hpa)

```bash
# 可以使用kubectl explain命令查看k8s API资源对象描述信息
[root@k8s-master ~]# kubectl explain rs

# 查看API资源列表
[root@k8s-master ~]# kubectl api-resources
```

### RS和RC

ReplicationController(RC)用来确保容器应用的副本数始终保持在用户定义的副本数，即如果有容器异常退出，会自动创建新的 Pod 来替代；而如果异常多出来的容器也会自动回收。

在新版本的Kubernetes中建议使用ReplicaSet来取代ReplicationController，ReplicaSet规则跟ReplicationController没有本质的不同，唯一区别是ReplicaSet支持`selector`选择器。

ReplicaSet也很少单独被使用，都是使用更高级的资源Deployment、DaemonSet、StatefulSet进行管理Pod，ReplicaSet主要用作Deployment协调创建、删除和更新Pod。

```bash
# RC实例，也是后面的rc-demo.yaml中的内容
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    app: nginx
  template:
    metadata:
      name: nginx
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
# RS实例，和RC配置的主要区别是多了个selector
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: gcr.io/google_samples/gb-frontend:v3
# 以上面RC实例配置为例
[root@k8s-master01 ~]# kubectl apply -f rc-demo.yaml 
replicationcontroller/nginx created
[root@k8s-master01 ~]# kubectl get po,rc
NAME                           READY   STATUS    RESTARTS        AGE
pod/nginx-5w6v6                1/1     Running   0               6s
pod/nginx-8265c                1/1     Running   0               6s

NAME                          DESIRED   CURRENT   READY   AGE
replicationcontroller/nginx   2         2         2       6s
# RC的Pod动态缩放
[root@k8s-master01 ~]# kubectl scale rc nginx --replicas=3
replicationcontroller/nginx scaled
# 可以看到正在扩容
[root@k8s-master01 ~]# kubectl get po,rc
NAME              READY   STATUS              RESTARTS   AGE
pod/nginx-5w6v6   1/1     Running             0          97s
pod/nginx-8265c   1/1     Running             0          97s
pod/nginx-jhsjr   0/1     ContainerCreating   0          3s

NAME                          DESIRED   CURRENT   READY   AGE
replicationcontroller/nginx   3         3         2       97s
# 扩容成功
[root@k8s-master01 ~]# kubectl get po,rc
NAME              READY   STATUS    RESTARTS   AGE
pod/nginx-5w6v6   1/1     Running   0          2m17s
pod/nginx-8265c   1/1     Running   0          2m17s
pod/nginx-jhsjr   1/1     Running   0          43s
# 缩容测试
[root@k8s-master01 ~]# kubectl scale rc nginx --replicas=1
replicationcontroller/nginx scaled
[root@k8s-master01 ~]# kubectl get po,rc
NAME              READY   STATUS    RESTARTS   AGE
pod/nginx-5w6v6   1/1     Running   0          4m26s

NAME                          DESIRED   CURRENT   READY   AGE
replicationcontroller/nginx   1         1         1       4m26s
```

### Deployment

用于部署无状态的服务，最常用的控制器。一般用于管理维护企业内部无状态的微服务，比如configserver、zuul、springboot。可以管理多个副本的Pod实现无缝迁移、自动扩容缩容、自动灾难恢复、一键回滚等功能。

Deployment是一种更高级别的 API 资源对象，为 Pods 和 ReplicaSets 提供声明式的更新能力。它以类似于 `kubectl rolling-update` 的方式更新其底层 ReplicaSet 及其 Pod。

Deployments 的典型用例：

- 创建 Deployment 将 ReplicaSet 上线.ReplicaSet 在后台创建 Pods.检查 ReplicaSet 的上线状态，查看其是否成功
- 通过更新 Deployment 的 PodTemplateSpec，声明 Pod 的新状态。新的 ReplicaSet 会被创建，Deployment 将 Pod 从旧 ReplicaSet 迁移到新 ReplicaSet。 每个新的 ReplicaSet 都会更新 Deployment 的修订版本
- 如果 Deployment 的当前状态不稳定，回滚到较早的 Deployment 版本。每次回滚都会更新 Deployment 的修订版本
- 扩大 Deployment 规模以承担更多负载
- 暂停 Deployment 以应用对 PodTemplateSpec 所作的多项修改，然后恢复其执行以启动新的上线版本
- 使用 Deployment 状态来判定上线过程是否出现停滞
- 清理较旧的不再需要的 ReplicaSet

#### 创建Deployment

Deployment创建一个ReplicaSet，该RS负责启动管理三个Pod。

```bash
# 创建yaml文件
[root@k8s-master01 ~]# vim nginx-deploy.yaml
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
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
[root@k8s-master01 ~]# kubectl apply -f nginx-deploy.yaml 
deployment.apps/nginx-deploy created
[root@k8s-master01 ~]# kubectl get deploy,rs,po
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deploy   3/3     3            3           58s

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-deploy-ff6655784   3         3         3       58s

NAME                               READY   STATUS    RESTARTS   AGE
pod/nginx-deploy-ff6655784-2jxbv   1/1     Running   0          58s
pod/nginx-deploy-ff6655784-b8gq8   1/1     Running   0          58s
pod/nginx-deploy-ff6655784-qj2k2   1/1     Running   0          58s
# 查看标签
[root@k8s-master01 ~]# kubectl get po --show-labels 
NAME                           READY   STATUS    RESTARTS   AGE   LABELS
nginx-deploy-ff6655784-2jxbv   1/1     Running   0          98s   app=nginx,pod-template-hash=ff6655784
nginx-deploy-ff6655784-b8gq8   1/1     Running   0          98s   app=nginx,pod-template-hash=ff6655784
nginx-deploy-ff6655784-qj2k2   1/1     Running   0          98s   app=nginx,pod-template-hash=ff6655784
# 扩缩容
[root@k8s-master01 ~]# kubectl scale deployment nginx-deploy --replicas=2
deployment.apps/nginx-deploy scaled
[root@k8s-master01 ~]# kubectl get deploy,rs,po
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deploy   2/2     2            2           2m18s

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-deploy-ff6655784   2         2         2       2m18s

NAME                               READY   STATUS    RESTARTS   AGE
pod/nginx-deploy-ff6655784-b8gq8   1/1     Running   0          2m18s
pod/nginx-deploy-ff6655784-qj2k2   1/1     Running   0          2m18s
# 查看pod详情
[root@k8s-master01 ~]# kubectl get po -owide
NAME                           READY   STATUS    RESTARTS   AGE     IP              NODE           NOMINATED NODE   READINESS GATES
nginx-deploy-ff6655784-b8gq8   1/1     Running   0          2m40s   172.25.92.66    k8s-master02   <none>           <none>
nginx-deploy-ff6655784-qj2k2   1/1     Running   0          2m40s   172.17.125.18   k8s-node01     <none>           <none>
# 访问nginx
[root@k8s-master01 ~]# curl 172.25.92.66
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</html>
[root@k8s-master01 ~]# curl 172.17.125.18
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</html>
```

说明:

- ReplicaSet 的名称始终被格式化为[DEPLOYMENT-NAME]-[RANDOM-STRING]。`RANDOM-STRING`随机生成。并使用`pod-template-hash=[RANDOM-STRING]`作为选择器和标签
- Deployment 控制器将`pod-template-hash=[RANDOM-STRING]`标签添加到 Deployment 创建或使用的ReplicaSet和Pod。此标签可确保 Deployment的子ReplicaSets不重叠，因此不可修改
- 注意Deployment、ReplicaSet和Pod三者的名称关系

```bash
[root@k8s-master01 ~]# kubectl get rs --show-labels 
NAME                     DESIRED   CURRENT   READY   AGE     LABELS
nginx-deploy-ff6655784   2         2         2       6m49s   app=nginx,pod-template-hash=ff6655784
[root@k8s-master01 ~]# kubectl get deploy,rs,po --show-labels 
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE    LABELS
deployment.apps/nginx-deploy   2/2     2            2           8m2s   app=nginx

NAME                                     DESIRED   CURRENT   READY   AGE    LABELS
replicaset.apps/nginx-deploy-ff6655784   2         2         2       8m2s   app=nginx,pod-template-hash=ff6655784

NAME                               READY   STATUS    RESTARTS   AGE    LABELS
pod/nginx-deploy-ff6655784-b8gq8   1/1     Running   0          8m2s   app=nginx,pod-template-hash=ff6655784
pod/nginx-deploy-ff6655784-qj2k2   1/1     Running   0          8m2s   app=nginx,pod-template-hash=ff6655784
```

#### 更新Deployment

Deployment 可确保在更新时仅关闭一定数量的 Pods.默认情况下，它确保至少 75% 所需 Pods 在运行(25%为容忍的最大不可用量)。更新时不会先删除旧的pod，而是先新建一个pod.新pod运行时，才会删除对应老的pod。一切的前提都是为了满足上述的条件。

备注: 如果需要更新Deployment，最好通过yaml文件更新，这样回滚到任何版本都非常便捷，而且更容易追溯。

```bash
# 方式一: 直接修改镜像[不推荐]
# 执行下面命令后修改对于镜像版本即可, 该方法不会记录命令,通过kubectl rollout history deployment/nginx-deploy 无法查询
[root@k8s-master01 ~]# kubectl edit deploy/nginx-deploy

# 方式二: 命令行更新image[可使用]，但也提示--record即将废弃
[root@k8s-master01 ~]# kubectl set image deploy/nginx-deploy nginx=nginx:1.18.0 --record
Flag --record has been deprecated, --record will be removed in the future
deployment.apps/nginx-deploy image updated
[root@k8s-master01 ~]# kubectl rollout history deployment nginx-deploy 
deployment.apps/nginx-deploy 
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl set image deploy/nginx-deploy nginx=nginx:1.18.0 --record=true
[root@k8s-master01 ~]# kubectl get deployments.apps -owide
NAME           READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES         SELECTOR
nginx-deploy   2/2     2            2           16m   nginx        nginx:1.18.0   app=nginx
# 方式三: 使用yaml文件更新版本
# 当前nginx 版本是1.16.1, 更新到1.18.0
# 先删除之前更新的deploy，在重新启动
[root@k8s-master01 ~]# kubectl delete -f nginx-deploy.yaml 
deployment.apps "nginx-deploy" deleted
[root@k8s-master01 ~]# kubectl apply -f nginx-deploy.yaml 
deployment.apps/nginx-deploy created
[root@k8s-master01 ~]# kubectl get deployments.apps -owide
NAME           READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES         SELECTOR
nginx-deploy   3/3     3            3           35s   nginx        nginx:1.16.1   app=nginx
# 拷贝一份yaml文件，二者重命名（老版保留以便有问题时回退），编辑新文件镜像为1.18.0
[root@k8s-master01 ~]# ll
total 8
-rw-r--r-- 1 root root 337 Dec 21 09:59 nginx-deploy-1161.yaml
-rw-r--r-- 1 root root 337 Dec 21 11:01 nginx-deploy-1180.yaml
[root@k8s-master01 ~]# cat nginx-deploy-1161.yaml 
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
[root@k8s-master01 ~]# cat nginx-deploy-1180.yaml 
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
[root@k8s-master01 ~]# kubectl apply -f nginx-deploy-1180.yaml --record
deployment.apps/nginx-deploy configured
# 如果正在更新, 可看到下面日志
[root@k8s-master01 ~]# kubectl rollout status deploy/nginx-deploy
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
[root@k8s-master01 ~]# kubectl rollout status deploy/nginx-deploy
deployment "nginx-deploy" successfully rolled out
```

#### 回滚Deployment

```bash
# 回滚到上一个版本
[root@k8s-master01 ~]# kubectl rollout undo deployment nginx-deploy
deployment.apps/nginx-deploy rolled back
[root@k8s-master01 ~]# kubectl get deploy,rs,po -owide
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES         SELECTOR
deployment.apps/nginx-deploy   3/3     3            3           11m   nginx        nginx:1.16.1   app=nginx

NAME                                     DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES         SELECTOR
replicaset.apps/nginx-deploy-79fccc485   0         0         0       7m46s   nginx        nginx:1.18.0   app=nginx,pod-template-hash=79fccc485
replicaset.apps/nginx-deploy-ff6655784   3         3         3       11m     nginx        nginx:1.16.1   app=nginx,pod-template-hash=ff6655784

NAME                               READY   STATUS    RESTARTS   AGE     IP              NODE           NOMINATED NODE   READINESS GATES
pod/nginx-deploy-ff6655784-c9gf9   1/1     Running   0          2m7s    172.25.92.69    k8s-master02   <none>           <none>
pod/nginx-deploy-ff6655784-xl4rx   1/1     Running   0          2m12s   172.27.14.228   k8s-node02     <none>           <none>
pod/nginx-deploy-ff6655784-zzpvm   1/1     Running   0          2m9s    172.17.125.22   k8s-node01     <none>           <none>

# 直接用Yaml文件回退[更新]为对应镜像, 比如 1180-->1161
[root@k8s-master01 ~]# kubectl apply -f nginx-deploy-1161.yaml --record
```

#### 回退历史版本

```bash
# 查看历史版本
[root@k8s-master01 ~]# kubectl rollout history deployment nginx-deploy 
deployment.apps/nginx-deploy 
REVISION  CHANGE-CAUSE
2         kubectl apply --filename=nginx-deploy.yaml --record=true
3         <none>
# 查看历史版本信息
[root@k8s-master01 ~]# kubectl rollout history deployment nginx-deploy --revision=2
deployment.apps/nginx-deploy with revision #2
Pod Template:
  Labels:       app=nginx
        pod-template-hash=79fccc485
  Annotations:  kubernetes.io/change-cause: kubectl apply --filename=nginx-deploy.yaml --record=true
  Containers:
   nginx:
    Image:      nginx:1.18.0
    Port:       80/TCP
    Host Port:  0/TCP
    Environment:        <none>
    Mounts:     <none>
  Volumes:      <none>
# 可以使用--to-revision参数指定回退到某个历史版本
[root@k8s-master01 ~]# kubectl rollout undo deployment nginx-deploy --to-revision=2
deployment.apps/nginx-deploy rolled back
[root@k8s-master01 ~]# kubectl get deploy,rs,po -owide
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES         SELECTOR
deployment.apps/nginx-deploy   3/3     3            3           14m   nginx        nginx:1.18.0   app=nginx

NAME                                     DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES         SELECTOR
replicaset.apps/nginx-deploy-79fccc485   3         3         3       10m   nginx        nginx:1.18.0   app=nginx,pod-template-hash=79fccc485
replicaset.apps/nginx-deploy-ff6655784   0         0         0       14m   nginx        nginx:1.16.1   app=nginx,pod-template-hash=ff6655784

NAME                               READY   STATUS    RESTARTS   AGE   IP              NODE           NOMINATED NODE   READINESS GATES
pod/nginx-deploy-79fccc485-dwhhr   1/1     Running   0          49s   172.25.92.70    k8s-master02   <none>           <none>
pod/nginx-deploy-79fccc485-kp2pd   1/1     Running   0          51s   172.17.125.23   k8s-node01     <none>           <none>
pod/nginx-deploy-79fccc485-z425l   1/1     Running   0          53s   172.27.14.229   k8s-node02     <none>           <none>
```

- **查看更新详情**

```bash
[root@k8s-master01 ~]# kubectl describe deployments.apps nginx-deploy 
Name:                   nginx-deploy
Namespace:              default
CreationTimestamp:      Wed, 12 Jan 2022 13:17:12 +0800
Labels:                 app=nginx
Annotations:            deployment.kubernetes.io/revision: 4
                        kubernetes.io/change-cause: kubectl apply --filename=nginx-deploy.yaml --record=true
Selector:               app=nginx
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge # 更新策略
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
NewReplicaSet:   nginx-deploy-79fccc485 (3/3 replicas created)
Events: # 可以看到之前的操作记录和更新记录
  Type    Reason             Age                    From                   Message
  ----    ------             ----                   ----                   -------
  Normal  ScalingReplicaSet  7m30s                  deployment-controller  Scaled up replica set nginx-deploy-ff6655784 to 1
  Normal  ScalingReplicaSet  7m27s                  deployment-controller  Scaled down replica set nginx-deploy-79fccc485 to 2
  Normal  ScalingReplicaSet  7m25s (x2 over 17m)    deployment-controller  Scaled up replica set nginx-deploy-ff6655784 to 3
  Normal  ScalingReplicaSet  7m23s (x3 over 7m27s)  deployment-controller  (combined from similar events): Scaled down replica set nginx-deploy-79fccc485 to 0
  Normal  ScalingReplicaSet  3m16s (x2 over 13m)    deployment-controller  Scaled up replica set nginx-deploy-79fccc485 to 1
  Normal  ScalingReplicaSet  3m14s (x2 over 13m)    deployment-controller  Scaled down replica set nginx-deploy-ff6655784 to 2
  Normal  ScalingReplicaSet  3m14s (x2 over 13m)    deployment-controller  Scaled up replica set nginx-deploy-79fccc485 to 2
  Normal  ScalingReplicaSet  3m12s (x2 over 13m)    deployment-controller  Scaled down replica set nginx-deploy-ff6655784 to 1
  Normal  ScalingReplicaSet  3m12s (x2 over 13m)    deployment-controller  Scaled up replica set nginx-deploy-79fccc485 to 3
  Normal  ScalingReplicaSet  3m10s (x2 over 10m)    deployment-controller  Scaled down replica set nginx-deploy-ff6655784 to 0
# 查看回滚状态
[root@k8s-master01 ~]# kubectl rollout status deploy nginx-deploy
deployment "nginx-deploy" successfully rolled out
# 历史记录
[root@k8s-master01 ~]# kubectl rollout history deploy nginx-deploy
deployment.apps/nginx-deploy
REVISION  CHANGE-CAUSE
3         <none>
4         kubectl set image deployment/nginx-deploy nginx=nginx:1.18.0 --record=true
```

***暂停与恢复**

```bash
# 暂停 deployment 的更新，暂停后除非恢复更新，否则操作不会生效，当需要多项更新时可以使用
[root@k8s-master01 ~]# kubectl rollout pause deployment nginx-deploy 
deployment.apps/nginx-deploy paused
[root@k8s-master01 ~]# kubectl set image deploy/nginx-deploy nginx=nginx:1.16.1 --record
Flag --record has been deprecated, --record will be removed in the future
deployment.apps/nginx-deploy image updated
[root@k8s-master01 ~]# kubectl get deployments.apps -owide
NAME           READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES         SELECTOR
nginx-deploy   3/3     0            3           25m   nginx        nginx:1.16.1   app=nginx
# 可以看到镜像确实是更新了，但RS和Pod在用的还是老版本的，也就是并未生效
[root@k8s-master01 ~]# kubectl get rs,pod -owide
NAME                                     DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES         SELECTOR
replicaset.apps/nginx-deploy-79fccc485   3         3         3       22m   nginx        nginx:1.18.0   app=nginx,pod-template-hash=79fccc485
replicaset.apps/nginx-deploy-ff6655784   0         0         0       26m   nginx        nginx:1.16.1   app=nginx,pod-template-hash=ff6655784

NAME                               READY   STATUS    RESTARTS   AGE   IP              NODE           NOMINATED NODE   READINESS GATES
pod/nginx-deploy-79fccc485-dwhhr   1/1     Running   0          12m   172.25.92.70    k8s-master02   <none>           <none>
pod/nginx-deploy-79fccc485-kp2pd   1/1     Running   0          12m   172.17.125.23   k8s-node01     <none>           <none>
pod/nginx-deploy-79fccc485-z425l   1/1     Running   0          12m   172.27.14.229   k8s-node02     <none>           <none>
# 接着进行下一项更新，配置CPU和内存资源
[root@k8s-master01 ~]# kubectl set resources deploy nginx-deploy -c nginx --limits=cpu=100m,memory=128Mi --requests=cpu=10m,memory=16Mi
deployment.apps/nginx-deploy resource requirements updated
# 恢复deployment的更新
[root@k8s-master01 ~]# kubectl rollout resume deployment nginx-deploy 
deployment.apps/nginx-deploy resumed
# 正在更新
[root@k8s-master01 ~]# kubectl get rs
NAME                     DESIRED   CURRENT   READY   AGE
nginx-deploy-79fccc485   1         1         1       28m
nginx-deploy-975d4fc74   3         3         2       5s
nginx-deploy-ff6655784   0         0         0       32m
# 更新完成后
[root@k8s-master01 ~]# kubectl get deploy,rs,po -owide
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES         SELECTOR
deployment.apps/nginx-deploy   3/3     3            3           32m   nginx        nginx:1.16.1   app=nginx

NAME                                     DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES         SELECTOR
replicaset.apps/nginx-deploy-79fccc485   0         0         0       28m   nginx        nginx:1.18.0   app=nginx,pod-template-hash=79fccc485
replicaset.apps/nginx-deploy-975d4fc74   3         3         3       25s   nginx        nginx:1.16.1   app=nginx,pod-template-hash=975d4fc74
replicaset.apps/nginx-deploy-ff6655784   0         0         0       32m   nginx        nginx:1.16.1   app=nginx,pod-template-hash=ff6655784

NAME                               READY   STATUS    RESTARTS   AGE   IP              NODE           NOMINATED NODE   READINESS GATES
pod/nginx-deploy-975d4fc74-6d92q   1/1     Running   0          25s   172.27.14.230   k8s-node02     <none>           <none>
pod/nginx-deploy-975d4fc74-7c7f2   1/1     Running   0          22s   172.17.125.24   k8s-node01     <none>           <none>
pod/nginx-deploy-975d4fc74-snpkb   1/1     Running   0          20s   172.25.92.71    k8s-master02   <none>           <none>
# 可以看到资源配额和镜像版本均已更新
[root@k8s-master01 ~]# kubectl describe deployments.apps nginx-deploy 
...
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:      nginx:1.16.1
    Port:       80/TCP
    Host Port:  0/TCP
    Limits:
      cpu:     100m
      memory:  128Mi
    Requests:
      cpu:        10m
      memory:     16Mi
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
...
```

***其他**

- .spec.revisionHistoryLimit：设置保留RS旧的revision的个数，设置为0的话，不保留历史数据
- .spec.minReadySeconds：可选参数，指定新创建的Pod在没有任何容器崩溃的情况下视为Ready最小的秒数，默认为0，即一旦被创建就视为可用。
- 滚动更新的策略：
  - .spec.strategy.type：更新deployment的方式，默认是RollingUpdate
    - RollingUpdate：滚动更新，可以指定maxSurge和maxUnavailable
      - maxUnavailable：指定在回滚或更新时最大不可用的Pod的数量，可选字段，默认25%，可以设置成数字或百分比，如果该值为0，那么maxSurge就不能0
      - maxSurge：可以超过期望值的最大Pod数，可选字段，默认为25%，可以设置成数字或百分比，如果该值为0，那么maxUnavailable不能为0
    - Recreate：重建，先删除旧的Pod，在创建新的Pod

```bash
# spec中的配置实例
spec:
  progressDeadlineSeconds: 600
  replicas: 2
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: nginx
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      ...
```

### StatefulSet

StatefulSet常用于部署有状态的且需要有序启动的应用程序，比如在生产环境中，可以部署ElasticSearch集群、MongoDB集群或者需要持久化的RabbitMQ集群、Redis集群、Kafka集群和ZooKeeper集群等。

StatefulSet 中的 Pod 拥有独一无二的身份标识。这个标识基于 StatefulSet 控制器分配给每个 Pod 的唯一顺序索引。Pod 的名称的形式为`<statefulset name>-<ordinal index>` 。例如: web的StatefulSet 拥有两个副本，所以它创建了两个 Pod: web-0和web-1。

和 Deployment 相同的是，StatefulSet 管理了基于相同容器定义的一组 Pod；但和 Deployment 不同的是，StatefulSet 为它们的每个 Pod 维护了一个固定的ID标识。这些 Pod 是基于相同的声明来创建的，但是不能相互替换: 无论怎么调度，每个 Pod 都有一个永久不变的ID标识。

StatefulSet创建的Pod一般使用Headless Service（无头服务）进行通信，和普通的Service的区别在于Headless Service没有ClusterIP，它使用的是Endpoint进行互相通信，Headless一般的格式为：statefulSetName-{0..N-1}.serviceName.namespace.svc.cluster.local。

- serviceName为Headless Service的名字，创建StatefulSet时，必须指定Headless Service名称
- 0..N-1为Pod所在的序号，从0开始到N-1
- statefulSetName为StatefulSet的名字
- namespace为服务所在的命名空间
- .cluster.local为Cluster Domain（集群域）

#### 使用场景

- 稳定的、唯一的网络标识符,即Pod重新调度后其PodName和HostName不变[当然IP是会变的]
- 稳定的、持久的存储,即Pod重新调度后还是能访问到相同的持久化数据,基于PVC实现
- 有序的、优雅的部署和缩放
- 有序的、自动的滚动更新

如上面,稳定意味着 Pod 调度或重调度的整个过程是有持久性的。

如果应用程序不需要任何稳定的标识符或有序的部署、删除或伸缩，则应该使用由一组无状态的副本控制器提供的工作负载来部署应用程序，比如使用 Deployment 或者 ReplicaSet 可能更适用于无状态应用部署需要。

***限制**

- 给定 Pod 的存储必须由 PersistentVolume(PV) 驱动基于所请求的 storage class 来提供,或者由管理员预先提供
- 删除或者收缩 StatefulSet 并不会删除它关联的存储卷.这样做是为了保证数据安全，它通常比自动清除 StatefulSet 所有相关的资源更有价值
- StatefulSet 当前需要 headless 服务来负责 Pod 的网络标识。需要先创建此服务。
- 当删除 StatefulSets 时,StatefulSet 不提供任何终止 Pod 的保证。为了实现 StatefulSet 中的 Pod 可以有序和优雅的终止，可以在删除之前将 StatefulSet 缩放为 0。
- 在默认 Pod 管理策略(OrderedReady) 时使用滚动更新，可能进入需要人工干预才能修复的损坏状态

***有序索引**

对于具有 N 个副本的 StatefulSet，StatefulSet 中的每个 Pod 将被分配一个整数序号，从 0 到 N-1，该序号在 StatefulSet 上是唯一的。

StatefulSet 中的每个 Pod 根据 StatefulSet 中的名称和 Pod 的序号来派生出它的主机名。组合主机名的格式为`<statefulset name>-<ordinal index>`。

#### 部署和扩缩保证

- 对于包含 N 个 副本的 StatefulSet，当部署 Pod 时，它们是依次创建的，顺序为 0~(N-1)
- 当删除 Pod 时，它们是逆序终止的,顺序为 (N-1)~0
- 在将缩放操作应用到 Pod 之前，它前面的所有 Pod 必须是 Running 和 Ready 状态
- 在 Pod 终止之前，所有的继任者必须完全关闭

StatefulSet 不应将 pod.Spec.TerminationGracePeriodSeconds 设置为 0。这种做法是不安全的，要强烈阻止。

***部署顺序**

在下面的 nginx 示例被创建后，会按照 web-0、web-1、web-2 的顺序部署三个 Pod。在 web-0 进入 Running 和 Ready 状态前不会部署 web-1。在 web-1 进入 Running 和 Ready 状态前不会部署 web-2。

如果 web-1 已经处于 Running 和 Ready 状态，而 web-2 尚未部署，在此期间发生了 web-0 运行失败，那么 web-2 将不会被部署，要等到 web-0 部署完成并进入 Running 和 Ready 状态后，才会部署 web-2。

***收缩顺序**

如果想将示例中的 StatefulSet 收缩为 replicas=1，首先被终止的是 web-2。在 web-2 没有被完全停止和删除前，web-1 不会被终止。当 web-2 已被终止和删除；但web-1 尚未被终止，如果在此期间发生 web-0 运行失败，那么就不会终止 web-1，必须等到 web-0 进入 Running 和 Ready 状态后才会终止 web-1。

```bash
# 查看StatefulSet说明
[root@k8s-master01 ~]# kubectl explain sts
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

#### STS实例

```bash
[root@k8s-master01 ~]# vim statefulset.yaml
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
  type: ClusterIP
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
  serviceName: nginx
  replicas: 3 # by default is 1
  template:
    metadata:
      labels:
        app: nginx # has to match .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 10   # 默认30秒
      containers:
      - name: nginx
        image: nginx:1.18.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
          name: http
[root@k8s-master01 ~]# kubectl apply -f statefulset.yaml 
service/nginx created
statefulset.apps/web created
[root@k8s-master01 ~]# kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   30d
nginx        ClusterIP   None         <none>        80/TCP    27s
[root@k8s-master01 ~]# kubectl get sts -owide
NAME   READY   AGE   CONTAINERS   IMAGES
web    3/3     63s   nginx        nginx:1.18.0
[root@k8s-master01 ~]# kubectl get po -owide
NAME    READY   STATUS    RESTARTS   AGE   IP              NODE           NOMINATED NODE   READINESS GATES
web-0   1/1     Running   0          83s   172.27.14.233   k8s-node02     <none>           <none>
web-1   1/1     Running   0          81s   172.17.125.28   k8s-node01     <none>           <none>
web-2   1/1     Running   0          79s   172.25.92.73    k8s-master02   <none>           <none>
# 创建busybox Pod进去查看三个web-Pod的域名信息
[root@k8s-master01 ~]# vi busybox.yaml
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
[root@k8s-master01 ~]# kubectl apply -f busybox.yaml 
pod/busybox created
# 进入busybox解析域名
[root@k8s-master01 ~]# kubectl exec busybox -it -- sh
/ # nslookup web-0.nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-0.nginx
Address 1: 172.27.14.233 web-0.nginx.default.svc.cluster.local
/ # nslookup web-1.nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-1.nginx
Address 1: 172.17.125.28 web-1.nginx.default.svc.cluster.local
/ # nslookup web-2.nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-2.nginx
Address 1: 172.25.92.73 web-2.nginx.default.svc.cluster.local
/ # exit
```

### DaemonSet

DaemonSet守护进程确保全部（或匹配的一部分）节点上部署一个Pod。当有新的节点加入集群时，也会为它们新增一个Pod，当有节点从集群移除时，这些Pod也会被回收。删除DaemonSet将会删除它创建的所有 Pod.

DaemonSet典型用法:

- 在每个节点上运行集群守护进程，例如 glusterd、ceph
- 在每个节点上运行日志守护进程，例如 fluentd、logstash
- 在每个节点上运行监控守护进程，例如 Prometheus Node Exporter、Flowmill、Sysdig 代理、collectd、Dynatrace OneAgent、AppDynamics 代理、Datadog 代理、New Relic 代理、Ganglia gmond 或者 Instana 代理

***DaemonSet实例**

```bash
[root@k8s-master01 ~]# kubectl explain ds
KIND:     DaemonSet
VERSION:  apps/v1

DESCRIPTION:
     DaemonSet represents the configuration of a daemon set.

FIELDS:
...
# 创建yaml
[root@k8s-master01 ~]# vim daemonset.yaml
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
      # 容忍在master节点运行
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
[root@k8s-master01 ~]# kubectl apply -f daemonset.yaml 
daemonset.apps/fluentd-elasticsearch created
[root@k8s-master01 ~]# kubectl get ds -owide
NAME                    DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE   CONTAINERS              IMAGES                                                            SELECTOR
fluentd-elasticsearch   5         5         0       5            0           <none>          11s   fluentd-elasticsearch   registry.cn-beijing.aliyuncs.com/google_registry/fluentd:v2.5.2   name=fluentd-elasticsearch
# 可见DaemonSet在每个节点上都有相应的pod
[root@k8s-master01 ~]# kubectl get po -owide
NAME                          READY   STATUS    RESTARTS   AGE    IP               NODE           NOMINATED NODE   READINESS GATES
fluentd-elasticsearch-blmjp   1/1     Running   0          2m9s   172.18.195.19    k8s-master03   <none>           <none>
fluentd-elasticsearch-t8bpk   1/1     Running   0          2m9s   172.17.125.29    k8s-node01     <none>           <none>
fluentd-elasticsearch-v8dwj   1/1     Running   0          2m9s   172.27.14.236    k8s-node02     <none>           <none>
fluentd-elasticsearch-w9wpb   1/1     Running   0          2m9s   172.25.92.74     k8s-master02   <none>           <none>
fluentd-elasticsearch-xrphv   1/1     Running   0          2m9s   172.25.244.193   k8s-master01   <none>           <none>
```

### Job

Job 负责仅执行一次的任务，它保证批处理任务的一个或多个 Pod 成功结束。

```bash
[root@k8s-master01 ~]# kubectl explain job
KIND:     Job
VERSION:  batch/v1

DESCRIPTION:
     Job represents the configuration of a single job.

FIELDS:
...
# 创建yaml
[root@k8s-master01 ~]# vi job-example.yaml
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
[root@k8s-master01 ~]# kubectl get po
NAME                      READY   STATUS      RESTARTS   AGE
daemonset-example-f57kg   1/1     Running     0          13m
pi-2nlj5                  0/1     Completed   0          4m
[root@k8s-master01 ~]# kubectl get job
NAME   COMPLETIONS   DURATION   AGE
pi     1/1           3m19s      4m15s
[root@k8s-master01 ~]# kubectl get po pi-2nlj5 -o wide
NAME       READY   STATUS      RESTARTS   AGE     IP              NODE        NOMINATED NODE   READINESS GATES
pi-2nlj5   0/1     Completed   0          4m54s   172.16.36.119   k8s-node1   <none>           <none>
# 查看日志可以看到job执行的结果
# 计算出了圆周率后1000位
```

![20201123180138](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20201123180138.png)

### CronJob

Cron Job 管理基于时间的 Job，即:

- 在给定时间点只运行一次
- 周期性地在给定时间点运行

典型的用法示例:

- 在给你写的时间点调度 Job 运行
- 创建周期性运行的 Job，例如: 数据库备份、发送邮件

```bash
[root@k8s-master01 ~]# kubectl explain cj
KIND:     CronJob
VERSION:  batch/v1

DESCRIPTION:
     CronJob represents the configuration of a single cron job.

FIELDS:
...
# 创建cronjob yaml文件
[root@k8s-master01 ~]# vi cronjob-example.yaml
apiVersion: batch/v1
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
          restartPolicy: OnFailure
[root@k8s-master01 ~]# kubectl apply -f cronjob-example.yaml
cronjob.batch/hello created
[root@k8s-master01 ~]# kubectl get cj
NAME    SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
hello   */1 * * * *   False     0        44s             69s
[root@k8s-master01 ~]# kubectl get po
NAME                   READY   STATUS      RESTARTS   AGE
hello-27366655-8mm5c   0/1     Completed   0          84s
hello-27366656-pjbv6   0/1     Completed   0          24s
# 查看输出日志
[root@k8s-master01 ~]# kubectl logs hello-27366655-8mm5c
Wed Jan 12 14:55:17 UTC 2022
Hello CronJob
[root@k8s-master01 ~]# kubectl logs hello-27366656-pjbv6
Wed Jan 12 14:56:15 UTC 2022
Hello CronJob
[root@k8s-master01 ~]# kubectl get job
NAME             COMPLETIONS   DURATION   AGE
hello-27366655   1/1           17s        2m29s
hello-27366656   1/1           16s        89s
hello-27366657   0/1           29s        29s
# 删除cronjob
[root@k8s-master01 ~]# kubectl delete cronjob hello
cronjob.batch "hello" deleted
[root@k8s-master01 ~]# kubectl get job
No resources found in default namespace.
```

### HPA

顾名思义，Horizontal Pod Autoscaling（Pod 水平自动缩放），可以基于 CPU 利用率自动伸缩 ReplicationController、Deployment、ReplicaSet 和 StatefulSet 中的 Pod 数量。除了 CPU 利用率，也可以基于其他应程序提供的`自定义度量指标`来执行自动扩缩。 Pod 自动扩缩不适用于无法扩缩的对象，比如 DaemonSet。

Pod 水平自动扩缩特性由 Kubernetes API 资源和控制器实现。资源决定了控制器的行为。 控制器会周期性地调整副本控制器或 Deployment 中的副本数量，以使得类似 Pod 平均 CPU 利用率、平均内存利用率这类观测到的度量值与用户所设定的目标值匹配。

下面是HPA工作机制模型图:

![20201222152148](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20201222152148.png)

HPA两个版本区别:

- autoscaling/v1：稳定版本，只支持 cpu
- autoscaling/v2beta：测试阶段
  - v2beta1(支持CPU、内存和自定义指标)
  - v2beta2(支持CPU、内存、自定义指标Custom和额外指标ExternalMetrics)

> Kubernetes目前（v1.23.0）autoscaling/v2版本已进入稳定版，支持CPU、内存、自定义指标Custom和额外指标ExternalMetrics

#### 指标

Pod 水平自动伸缩器的实现是一个控制回路，由控制器管理器的`--horizontal-pod-autoscaler-sync-period`参数指定周期（默认值为 15 秒）。

每个周期内，控制器管理器根据每个`HorizontalPodAutoscaler`定义中指定的指标查询资源利用率。控制器管理器可以从资源度量指标 API（按 Pod 统计的资源用量）和自定义度量指标 API（其他指标）获取度量值。

- 对于按 Pod 统计的资源指标（如 CPU），控制器从资源指标 API 中获取每一个`HorizontalPodAutoscaler`指定的 Pod 的度量值，如果设置了目标使用率，控制器获取每个 Pod 中的容器资源使用情况，并计算资源使用率。 如果设置了 target 值，将直接使用原始数据（不再计算百分比）。 接下来，控制器根据平均的资源使用率或原始值计算出扩缩的比例，进而计算出目标副本数。
- 如果 Pod 使用自定义指标，控制器机制与资源指标类似，区别在于自定义指标只使用原始值，而不是使用率。
- 如果 Pod 使用对象指标和外部指标（每个指标描述一个对象信息）。这个指标将直接根据目标设定值相比较，并生成一个上面提到的扩缩比例。在`autoscaling/v2beta2`版本 API 中，这个指标也可以根据 Pod 数量平分后再计算。

通常情况下，控制器将从一系列的聚合API（metrics.k8s.io、custom.metrics.k8s.io 和 external.metrics.k8s.io）中获取度量值。`metrics.k8s.io` API 通常由 Metrics 服务器（需要提前启动）提供。

#### HPA使用

1.使用kubectl

```bash
# 自动伸缩deploy/php-apache，目标CPU使用率50%，deploy副本数在1~10之间
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
```

2.使用yaml创建

```bash
# 同上
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

```bash
# 集群所有可调度Pod的节点先获取要使用的hpa示例镜像
docker pull registry.cn-beijing.aliyuncs.com/google_registry/hpa-example
```

```bash
# yaml
[root@k8s-master01 ~]# vim php-apache.yaml
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
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m
---
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
[root@k8s-master01 ~]# kubectl apply -f php-apache.yaml 
deployment.apps/php-apache created
service/php-apache created
[root@k8s-master01 ~]# kubectl get deployments.apps 
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
php-apache   1/1     1            1           9s
# 创建HPA
# 当pod中CPU使用率达50%就扩容.最小1个,最大10个
[root@k8s-master01 ~]# kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
horizontalpodautoscaler.autoscaling/php-apache autoscaled
[root@k8s-master01 ~]# kubectl get hpa
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/50%    1         10        1          18s
```

***压测php-apache**

```bash
# 创建压测Pod并进入, 先不要退出
[root@k8s-master01 ~]# kubectl run -it load-test --image=busybox /bin/sh
/ # 
# 另起一窗口可查看到pod在运行
[root@k8s-master01 ~]# kubectl get po | grep load
load-test                     1/1     Running   0             26s
# 在load-test继续中操作，连续访问php-apache的Service
/ # while true; do wget -q -O- http://php-apache.default.svc.cluster.local; done
OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!
# 一分钟左右，在另外一个窗口可看到，负载飙升, 副本数增多
[root@k8s-master01 ~]# kubectl get hpa
NAME         REFERENCE               TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   254%/50%   1         10        3          21m
# 自动扩容
[root@k8s-master01 ~]# kubectl get deploy,rs,po
NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/php-apache   6/6     6            6           23m

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/php-apache-7db9c95858   6         6         6       23m

NAME                              READY   STATUS    RESTARTS      AGE
pod/php-apache-7db9c95858-6wrp2   1/1     Running   0             32s
pod/php-apache-7db9c95858-7dbjv   1/1     Running   0             10m
pod/php-apache-7db9c95858-cfwrk   1/1     Running   0             32s
pod/php-apache-7db9c95858-ksbgh   1/1     Running   0             47s
pod/php-apache-7db9c95858-m9mtn   1/1     Running   0             47s
pod/php-apache-7db9c95858-znczt   1/1     Running   0             32s

# 停止压测

# Ctrl+C停止在跑的循环
/ # while true; do wget -q -O- http://php-apache.default.svc.cluster.local; done
OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!.<省略很多输出>..!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!^C

# 停止压测后负载降下来
[root@k8s-master01 ~]# kubectl get hpa
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/50%    1         10        6          24m

# 再过几分钟副本数也降了
[root@k8s-master01 ~]# kubectl get hpa
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/50%    1         10        1          29m
[root@k8s-master01 ~]# kubectl get deploy,rs,po
NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/php-apache   1/1     1            1           31m

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/php-apache-7db9c95858   1         1         1       31m

NAME                              READY   STATUS    RESTARTS      AGE
pod/php-apache-7db9c95858-7dbjv   1/1     Running   0             18m
```

> 说明:停止压测，pod数量也不会立即降下来.而是过段时间后才会慢慢降下来.这是为了防止由于网络原因或者间歇性流量突增、突降，导致pod回收太快后面流量上来后Pod数量不够.

***测试V2版本其他指标**

```bash
[root@k8s-master01 ~]# vim hpa-v2.yaml

```
