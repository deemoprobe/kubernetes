# Kubernetes基础之调度策略

## 1. 定向调度(nodeSelector)

Kubernetes上kube-scheduler负责pod调度，通过内置算法实现最佳节点的调度，当然也可以指定调度的节点

```shell
# 给k8s-node1节点打上test标签
[root@k8s-master manifests]# kubectl label nodes k8s-node1 zone=test
node/k8s-node1 labeled
# 查看node的标签
[root@k8s-master ~]# kubectl get nodes k8s-node1 --show-labels
NAME        STATUS   ROLES   AGE   VERSION   LABELS
k8s-node1   Ready    node    13d   v1.19.3   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-node1,kubernetes.io/os=linux,node-role.kubernetes.io/node=,zone=test
# 或者从描述里查看
[root@k8s-master manifests]# kubectl describe node k8s-node1
Name:               k8s-node1
Roles:              node
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=k8s-node1
                    kubernetes.io/os=linux
                    node-role.kubernetes.io/node=
                    zone=test
# 给pod加上定向调度设置
[root@k8s-master manifests]# vi nginx-deploy.yaml
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
      nodeSelector:
        zone: test
[root@k8s-master manifests]# kubectl apply -f nginx-deploy.yaml
deployment.apps/nginx-deploy created
# 查看到所有的pod均部署在k8s-node1上
[root@k8s-master manifests]# kubectl get pods -o wide
NAME                          READY   STATUS    RESTARTS   AGE   IP              NODE        NOMINATED NODE   READINESS GATES
nginx-deploy-79bf6fcf-5b7v6   1/1     Running   0          11s   172.16.36.126   k8s-node1   <none>           <none>
nginx-deploy-79bf6fcf-d4hz6   1/1     Running   0          11s   172.16.36.68    k8s-node1   <none>           <none>
nginx-deploy-79bf6fcf-thbpt   1/1     Running   0          11s   172.16.36.127   k8s-node1   <none>           <none>
```

当然也可以通过`kubectl get nodes k8s-node1 --show-labels`查到的系统标签进行定向调度

## 2. 亲和与反亲和调度

定向调度比较是一种强制分配的方式进行pod调度，推荐使用亲和性调度代替定向调度，亲和性调度有下面两种表达：

- requiredDuringSchedulingIgnoredDuringExecution: hard(硬限制)，严格执行，满足规则调度，否则不调度
- preferredDuringSchedulingIgnoredDuringExecution：soft(软限制)，尽力执行，优先满足规则调度，多个规则可用权重来决定先执行哪一个

OPerator参数：

- In：label的值在某个列表中
- NotIn：label的值不在某个列表中
- Gt：label的值大于某个值
- Lt：label的值小于某个值
- Exists：某个label存在
- DoesNotExist：某个label不存在

## 3. node亲和调度(nodeAffinity)

Note: 支持的operator操作： In, NotIn, Exists, DoesNotExist, Gt, Lt. 其中，NotIn and DoesNotExist用于实现反亲和性。

Note: weight范围1-100。这个涉及调度器的优选打分过程，每个node的评分都会加上这个weight，最后bind最高的node。

```shell
# 第一个规则限制只运行在amd64架构的节点上，第二个规则是尽量调度到在k8s-node1节点上
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: beta.kubernetes.io/arch
            operator: In
            values:
            - amd64
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
            - k8s-node1
  containers:
  - name: with-node-affinity
    image: k8s.gcr.io/pause:2.0
```

## 4. pod亲和和反亲和调度

Pod的亲和性与反亲和性是基于Node节点上已经运行pod的标签(而不是节点上的标签)决定的，从而约束哪些节点适合调度pod。

规则是：如果X已经运行了一个或多个符合规则Y的pod，则此pod应该在X中运行(如果是反亲和的情况下，则不应该在X中运行）。当然pod必须处在同一名称空间，不然亲和性/反亲和性无作用。  
X是一个拓扑域，可以使用topologyKey来表示它，topologyKey 的值是node节点标签的键以便系统用来表示这样的拓扑域。当然这里也有个隐藏条件，就是node节点标签的键值相同时，才是在同一拓扑域中；如果只是节点标签名相同，但是值不同，那么也不在同一拓扑域。

Pod的亲和性/反亲和性调度是根据拓扑域来界定调度的，而不是根据node节点。

Pod: 支持的operator操作：In, NotIn, Exists, DoesNotExist, Gt, Lt.

```shell
# 创建参照deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-flag
  labels:
    seccurity: s1
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx
# 亲和反亲和配置实例
apiVersion: apps/v1
kind: Deployment
metadata:
  name: affinity-all
  labels:
    app: affinity-all
spec:
  containers:
  - name: affinity-all
    image: k8s.gcr.io/pause:2.0
  affinity:
    # pod亲和性
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          # 由于是Pod亲和性/反亲和性；因此这里匹配规则写的是Pod的标签信息
          matchExpressions:
          - key: security
            operator: In
            values:
            - s1
        # 拓扑域
        topologyKey: disk-type
    # pod反亲和性
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          # 由于是Pod亲和性/反亲和性；因此这里匹配规则写的是Pod的标签信息
          matchExpressions:
          - key: app
            operator: In
            values:
            - nginx
        # 拓扑域
        topologyKey: kubernetes.io/hostname
```

上面创建的deployment应满足下面规则：

- 与security=s1的pod为同一种disk-type(同一种磁盘的拓扑域)
- 不与app=nginx的pod调度在同一node节点上

## 5. 污点和容忍(Taints和Tolerations)

Taint需要和Toleration配合使用，让pod避开某些节点，除非pod创建时声明容忍策略，否则不会在有污点的节点上运行。

```shell
# 为k8s-node1设置不能调度的污点
[root@k8s-master manifests]# kubectl taint nodes k8s-node1 test=node1:NoSchedule
# 如果创建pod时设置容忍策略，则该pod能够（不是必须）被分配到该节点，具体能不能分配到该节点上由分配算法决定
# 常见的容忍配置
tolerations:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "NoSchedule"
---
tolerations:
- key: "key"
  operator: "Exists"
  effect: "NoSchedule"
---
tolerations:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "NoExecute"
  tolerationSeconds: 3600

# 在yaml文件中的位置
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test
  labels:
    app: test
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: test
    spec:
      containers:
      - name: test
        image: nginx
      tolerations:
      - key: "test"
        operator: "Exists"
        effect: "NoSchedule"
```
