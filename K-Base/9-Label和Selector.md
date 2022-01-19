# Kubernetes基础之Label和Selector

- Label：对k8s中各种资源进行分类、分组，添加一个属性标签
- Selector：通过一个过滤的语法查找到对应标签的资源

![labelselector](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/labelselector.png)

> 这个deployment将启动3个Pod并监视始终要有3个Pod处于活跃状态，这个deployment管控Pod的方式就是通过标签`app: nginx`进行匹配；以标签为`app: nginx`的目标Service也是如此，通过选择标签进行对应标签Pod服务的发现。

## Label

Label（标签）是附加到 Kubernetes 对象（比如 Pods）上的`key: value`键值对。标签可以在创建时附加到对象，随后可以随时添加和修改。每个对象都可以定义一组键/值标签。

- 标签键值对区分大小写
- 标签键值对必须以字母数字字符 ([a- z0-9A -Z]) 开头和结尾
- 标签键值可以包含短线(-)、下划线 (_)、点 (.)
- 标签键不能为空，值可以
- 标签键支持可选前缀，以斜杠 (/) 分隔
- 标签键（无前缀）和值必须为 63 个字符或更少

标签示例：

- release: stable, release: beta
- environment: dev, environment: test, environment: prod
- tier: frontend, tier: backend, tier: cache
- partition: customerA, partition: customerB
- track: daily, track: weekly

```bash
# 常见配置方式
apiVersion: xxx
metadata:
  labels:
    key1: value1
    key2: value2
...

# 命令行用法
Usage:
  kubectl label [--overwrite] (-f FILENAME | TYPE NAME) KEY_1=VAL_1 ... KEY_N=VAL_N [--resource-version=version]

# 默认情况下，节点都会预设下面的标签
beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-node01,kubernetes.io/os=linux

# 创建实例sts和service
[root@k8s-master01 ~]# mkdir labelselector
[root@k8s-master01 ~]# cd labelselector/
[root@k8s-master01 labelselector]# vim nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels: # has to match .spec.template.metadata.labels
      app: nginx
  template:
    metadata:
      labels:
        app: nginx # has to match .spec.selector.matchLabels
    spec:
      containers:
      - name: nginx
        image: nginx:1.18.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
[root@k8s-master01 labelselector]# vim my-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector: 
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
[root@k8s-master01 labelselector]# kubectl apply -f .
service/my-service created
deployment.apps/nginx-deployment created
[root@k8s-master01 labelselector]# kubectl get deployments.apps,po,svc
NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deployment   3/3     3            3           27s

NAME                                   READY   STATUS    RESTARTS   AGE
pod/nginx-deployment-79fccc485-pj665   1/1     Running   0          26s
pod/nginx-deployment-79fccc485-pxp49   1/1     Running   0          27s
pod/nginx-deployment-79fccc485-vw6np   1/1     Running   0          26s

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/my-service   ClusterIP   10.109.60.44   <none>        80/TCP    27s
```

### 查询Label

```bash
[root@k8s-master01 labelselector]# kubectl get deployments.apps --show-labels
NAME               READY   UP-TO-DATE   AVAILABLE   AGE   LABELS
nginx-deployment   3/3     3            3           80s   app=nginx
[root@k8s-master01 labelselector]# kubectl get po --show-labels
NAME                               READY   STATUS    RESTARTS   AGE     LABELS
nginx-deployment-79fccc485-pj665   1/1     Running   0          2m26s   app=nginx,pod-template-hash=79fccc485
nginx-deployment-79fccc485-pxp49   1/1     Running   0          2m27s   app=nginx,pod-template-hash=79fccc485
nginx-deployment-79fccc485-vw6np   1/1     Running   0          2m26s   app=nginx,pod-template-hash=79fccc485
# 也可查看SELECTOR
[root@k8s-master01 labelselector]# kubectl get deployments.apps -owide
NAME               READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES         SELECTOR
nginx-deployment   3/3     3            3           73s   nginx        nginx:1.18.0   app=nginx
```

### 新增Label

```bash
[root@k8s-master01 labelselector]# kubectl label deployments.apps nginx-deployment environment=test
deployment.apps/nginx-deployment labeled
[root@k8s-master01 labelselector]# kubectl get deployments.apps --show-labels
NAME               READY   UP-TO-DATE   AVAILABLE   AGE     LABELS
nginx-deployment   3/3     3            3           6m59s   app=nginx,environment=test
```

### 修改Label

```bash
# 修改已存在的标签
[root@k8s-master01 labelselector]# kubectl label --overwrite deployments.apps nginx-deployment environment=uat
deployment.apps/nginx-deployment unlabeled
[root@k8s-master01 labelselector]# kubectl get deployments.apps --show-labels
NAME               READY   UP-TO-DATE   AVAILABLE   AGE    LABELS
nginx-deployment   3/3     3            3           8m8s   app=nginx,environment=uat
```

### 删除Label

```bash
[root@k8s-master01 labelselector]# kubectl label deployments.apps nginx-deployment environment-
deployment.apps/nginx-deployment unlabeled
[root@k8s-master01 labelselector]# kubectl get deployments.apps --show-labels
NAME               READY   UP-TO-DATE   AVAILABLE   AGE   LABELS
nginx-deployment   3/3     3            3           12m   app=nginx

# 查看集群节点的标签
[root@k8s-master01 labelselector]# kubectl get nodes --show-labels 
NAME           STATUS   ROLES                  AGE   VERSION   LABELS
k8s-master01   Ready    control-plane,master   36d   v1.23.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-master01,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node-role.kubernetes.io/master=,node.kubernetes.io/exclude-from-external-load-balancers=
k8s-master02   Ready    control-plane,master   36d   v1.23.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-master02,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node-role.kubernetes.io/master=,node.kubernetes.io/exclude-from-external-load-balancers=
k8s-master03   Ready    control-plane,master   36d   v1.23.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-master03,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node-role.kubernetes.io/master=,node.kubernetes.io/exclude-from-external-load-balancers=
k8s-node01     Ready    node                   36d   v1.23.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-node01,kubernetes.io/os=linux,node-role.kubernetes.io/node=
k8s-node02     Ready    node                   36d   v1.23.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-node02,kubernetes.io/os=linux,node-role.kubernetes.io/node=
# 比如节点角色标签node-role.kubernetes.io/control-plane=,node-role.kubernetes.io/node=
# 删除k8s-node02的节点角色标签
[root@k8s-master01 labelselector]# kubectl label nodes k8s-node02 node-role.kubernetes.io/node-
node/k8s-node02 unlabeled
# Role变为<none>
[root@k8s-master01 labelselector]# kubectl get nodes --show-labels 
NAME           STATUS   ROLES                  AGE   VERSION   LABELS
k8s-master01   Ready    control-plane,master   36d   v1.23.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-master01,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node-role.kubernetes.io/master=,node.kubernetes.io/exclude-from-external-load-balancers=
k8s-master02   Ready    control-plane,master   36d   v1.23.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-master02,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node-role.kubernetes.io/master=,node.kubernetes.io/exclude-from-external-load-balancers=
k8s-master03   Ready    control-plane,master   36d   v1.23.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-master03,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node-role.kubernetes.io/master=,node.kubernetes.io/exclude-from-external-load-balancers=
k8s-node01     Ready    node                   36d   v1.23.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-node01,kubernetes.io/os=linux,node-role.kubernetes.io/node=
k8s-node02     Ready    <none>                 36d   v1.23.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-node02,kubernetes.io/os=linux
```

## Selector

Selector（标签选择器）主要用于资源的匹配，只有符合条件的资源才会被调用或使用，可以使用该方式对集群中的各类资源进行分配。

Kubernetes API目前支持两种类型的Selector：基于等值的 和 基于集合的。Selector可以由逗号分隔的多个需求组成。在多个需求的情况下，必须满足所有要求，因此逗号分隔符充当逻辑与（&&）运算符。

### 等值需求

基于等值或基于不等值 的需求允许按标签键和值进行过滤。 匹配对象必须满足所有指定的标签约束，尽管它们也可能具有其他标签。 可接受的运算符有`=`、`==`和`!=`三种。 前两个表示相等（并且只是同义词），而后者表示不相等。

### 集合需求

基于集合的标签需求允许你通过一组值来过滤键。支持三种操作符：`in`、`notin`和`exists`(只可以用在键标识符上)。

```bash
# --selector参数进行标签的选择，可简写为-l
[root@k8s-master01 labelselector]# kubectl get deployments.apps --selector app=nginx
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           14m
# 新创建一个deployment，作为对照
[root@k8s-master01 labelselector]# kubectl create deployment --image=nginx other
deployment.apps/other created
[root@k8s-master01 labelselector]# kubectl get deployments.apps --show-labels -owide
NAME               READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES         SELECTOR    LABELS
nginx-deployment   3/3     3            3           17m   nginx        nginx:1.18.0   app=nginx   app=nginx
other              1/1     1            1           58s   nginx        nginx          app=other   app=other
# 等值查询，并在查询后展示标签
[root@k8s-master01 labelselector]# kubectl get deployments.apps -l app=other --show-labels
NAME    READY   UP-TO-DATE   AVAILABLE   AGE    LABELS
other   1/1     1            1           2m6s   app=other
# 等值反向查询
[root@k8s-master01 labelselector]# kubectl get deployments.apps -l app!=other --show-labels
NAME               READY   UP-TO-DATE   AVAILABLE   AGE   LABELS
nginx-deployment   3/3     3            3           18m   app=nginx
# 集合查询
[root@k8s-master01 labelselector]# kubectl get deployments.apps -l 'app in (other,nginx)' --show-labels 
NAME               READY   UP-TO-DATE   AVAILABLE   AGE     LABELS
nginx-deployment   3/3     3            3           21m     app=nginx
other              1/1     1            1           5m11s   app=other
# 再创建一个对照
[root@k8s-master01 labelselector]# vim busybox.yaml
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
[root@k8s-master01 labelselector]# kubectl apply -f busybox.yaml
pod/busybox created
[root@k8s-master01 labelselector]# kubectl get po --show-labels
NAME                               READY   STATUS    RESTARTS   AGE   LABELS
busybox                            1/1     Running   0          17s   <none>
nginx-deployment-79fccc485-pj665   1/1     Running   0          27m   app=nginx,pod-template-hash=79fccc485
nginx-deployment-79fccc485-pxp49   1/1     Running   0          27m   app=nginx,pod-template-hash=79fccc485
nginx-deployment-79fccc485-vw6np   1/1     Running   0          27m   app=nginx,pod-template-hash=79fccc485
other-657995d88d-pcf7h             1/1     Running   0          11m   app=other,pod-template-hash=657995d88d
[root@k8s-master01 labelselector]# kubectl label po busybox name=busybox
pod/busybox labeled
[root@k8s-master01 labelselector]# kubectl get po --show-labels
NAME                               READY   STATUS    RESTARTS   AGE   LABELS
busybox                            1/1     Running   0          61s   name=busybox
nginx-deployment-79fccc485-pj665   1/1     Running   0          28m   app=nginx,pod-template-hash=79fccc485
nginx-deployment-79fccc485-pxp49   1/1     Running   0          28m   app=nginx,pod-template-hash=79fccc485
nginx-deployment-79fccc485-vw6np   1/1     Running   0          28m   app=nginx,pod-template-hash=79fccc485
other-657995d88d-pcf7h             1/1     Running   0          11m   app=other,pod-template-hash=657995d88d
# 集合反向查询
[root@k8s-master01 labelselector]# kubectl get po -l 'app notin (nginx,other)' --show-labels
NAME      READY   STATUS    RESTARTS   AGE     LABELS
busybox   1/1     Running   0          4m49s   name=busybox

# 删除匹配了相应标签的资源
[root@k8s-master01 labelselector]# kubectl delete deployments.apps -l app=other
deployment.apps "other" deleted
[root@k8s-master01 labelselector]# kubectl get deployments.apps
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           33m
```
