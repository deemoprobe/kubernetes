# Kubernetes存储之ConfigMap

ConfigMap 是一种 API 对象,用来将非机密性的数据保存到健值对中.使用时可以用作环境变量/命令行参数或者存储卷中的配置文件.提供向容器内注入信息的功能.

ConfigMap 将环境配置信息和容器镜像解耦,便于应用配置的修改.当你需要储存机密信息时可以使用 Secret 对象.

备注: ConfigMap 并不提供保密或者加密功能.如果你想存储的数据是机密的,请使用 Secret;或者使用其他第三方工具来保证数据的私密性.而不是用 ConfigMap.

## 1. ConfigMap创建方式

### 1.1. 通过目录创建

`--from-file`指定目录下所有文件都作为ConfigMap中的键值对来使用, 键为文件名, 值为文件内容

```shell
# 进入自建的目录
[root@k8s-master volume]# pwd
/app/kubernetes/volume
# 创建配置文件目录并进入
[root@k8s-master volume]# mkdir configmap;cd configmap
[root@k8s-master configmap]# ll
total 8
-rw-r--r-- 1 root root 158 Dec 14 20:28 game.properties
-rw-r--r-- 1 root root  83 Dec 14 20:28 ui.properties
# 创建配置文件1-(game.properties)
[root@k8s-master configmap]# cat game.properties
enemies=aliens
lives=3
enemies.cheat=true
enemies.cheat.level=noGoodRotten
secret.code.passphrase=UUDDLRLRBABAs
secret.code.allowed=true
secret.code.lives=30
# 创建配置文件2-(ui.properties)
[root@k8s-master configmap]# cat ui.properties
color.good=purple
color.bad=yellow
allow.textmode=true
how.nice.to.look=fairlyNice
# 创建ConfigMap并查看,指向存放配置文件的目录
[root@k8s-master volume]# kubectl create configmap game-config --from-file=/app/kubernetes/volume/configmap
configmap/game-config created
[root@k8s-master volume]# kubectl get cm
NAME          DATA   AGE
game-config   2      12s
# 查看一下这些数据是什么
[root@k8s-master volume]# kubectl get cm game-config -o yaml
apiVersion: v1
# 两个DATA
data:
  game.properties: |
    enemies=aliens
    lives=3
    enemies.cheat=true
    enemies.cheat.level=noGoodRotten
    secret.code.passphrase=UUDDLRLRBABAs
    secret.code.allowed=true
    secret.code.lives=30
  ui.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
    how.nice.to.look=fairlyNice
kind: ConfigMap
metadata:
  creationTimestamp: "2020-12-15T01:35:21Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:game.properties: {}
        f:ui.properties: {}
    manager: kubectl-create
    operation: Update
    time: "2020-12-15T01:35:21Z"
  name: game-config
  namespace: default
  resourceVersion: "326707"
  selfLink: /api/v1/namespaces/default/configmaps/game-config
  uid: 38a78de2-8e9a-4219-bc84-7da1a29515ce
# 也可以通过下面命令查看
[root@k8s-master volume]# kubectl describe cm game-config
Name:         game-config
Namespace:    default
Labels:       <none>
Annotations:  <none>
# 两个DATA
Data
====
game.properties:
----
enemies=aliens
lives=3
enemies.cheat=true
enemies.cheat.level=noGoodRotten
secret.code.passphrase=UUDDLRLRBABAs
secret.code.allowed=true
secret.code.lives=30

ui.properties:
----
color.good=purple
color.bad=yellow
allow.textmode=true
how.nice.to.look=fairlyNice

Events:  <none>
```

### 1.2. 通过文件创建

`--from-file`指定对应配置文件, 可以多次使用引入不同的配置文件

```shell
# 指向为之前创建的具体文件
[root@k8s-master volume]# kubectl create configmap game-config-file --from-file=/app/kubernetes/volume/configmap/game.properties 
configmap/game-config-file created
[root@k8s-master volume]# kubectl get cm
NAME               DATA   AGE
game-config        2      20m
game-config-file   1      7s
# 查看DATA信息
[root@k8s-master volume]# kubectl get cm game-config-file -o yaml
apiVersion: v1
# 一个DATA
data:
  game.properties: |
    enemies=aliens
    lives=3
    enemies.cheat=true
    enemies.cheat.level=noGoodRotten
    secret.code.passphrase=UUDDLRLRBABAs
    secret.code.allowed=true
    secret.code.lives=30
kind: ConfigMap
metadata:
  creationTimestamp: "2020-12-15T01:55:14Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:game.properties: {}
    manager: kubectl-create
    operation: Update
    time: "2020-12-15T01:55:14Z"
  name: game-config-file
  namespace: default
  resourceVersion: "329421"
  selfLink: /api/v1/namespaces/default/configmaps/game-config-file
  uid: 5d94a64e-a338-4879-90aa-157bb6233c26
# 当然下面方式查看亦可,只是通过yaml形式输出更为详细一些
[root@k8s-master volume]# kubectl describe cm game-config-file
Name:         game-config-file
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
game.properties:
----
enemies=aliens
lives=3
enemies.cheat=true
enemies.cheat.level=noGoodRotten
secret.code.passphrase=UUDDLRLRBABAs
secret.code.allowed=true
secret.code.lives=30

Events:  <none>
```

### 1.3. 通过命令行创建

`--from-literal`传递配置参数, 可多次使用

```shell
# 指定两个字段, 通过命令写入配置
[root@k8s-master volume]# kubectl create configmap special-config --from-literal=special.how=very --from-literal="special.type=charm"
configmap/special-config created
[root@k8s-master volume]# kubectl get configmap special-config
NAME             DATA   AGE
special-config   2      14s
[root@k8s-master volume]# kubectl describe cm special-config
Name:         special-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
special.how:
----
very
special.type:
----
charm
Events:  <none>

```

### 1.4. 通过YAML文件创建

```shell
# 创建文件
[root@k8s-master volume]# vi configmap.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: configmap-demo
data:
  # 类属性键；每一个键都映射到一个简单的值
  player_initial_lives: "3"
  ui_properties_file_name: 'user-interface.properties'
  # 类文件键
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5
  user-interface.properties: |
    color.good=green
    color.bad=yellow
[root@k8s-master volume]# kubectl apply -f configmap.yaml 
configmap/configmap-demo created
[root@k8s-master volume]# kubectl get cm configmap-demo
NAME               DATA   AGE
configmap-demo     4      6s
[root@k8s-master volume]# kubectl describe cm configmap-demo
Name:         configmap-demo
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
game.properties:
----
enemy.types=aliens,monsters
player.maximum-lives=5

player_initial_lives:
----
3
ui_properties_file_name:
----
user-interface.properties
user-interface.properties:
----
color.good=green
color.bad=yellow

Events:  <none>
```

## 2. Pod中使用ConfigMap

在Pod中使用configmap配置信息

### 2.1. 查看当前的configmap

```shell
[root@k8s-master volume]# kubectl get cm
NAME               DATA   AGE
configmap-demo     4      4m25s
game-config        2      39m
game-config-file   1      19m
special-config     2      9m2s
```

### 2.2. 用configmap来代替环境变量

```shell
# 创建Pod
[root@k8s-master storage]# vi pod_configmap_env.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: pod-configmap-env
spec:
  containers:
  - name: myapp
    image: registry.cn-beijing.aliyuncs.com/google_registry/myapp:v1
    command: ["/bin/sh", "-c", "env"]
    ### 引用方式1
    env:
    - name: SPECAIL_HOW_KEY
      valueFrom:
        configMapKeyRef:
          name: special-config   ### 这个name的值来自 ConfigMap
          key: special.how       ### 这个key的值为需要取值的键
    - name: SPECAIL_TPYE_KEY
      valueFrom:
        configMapKeyRef:
          name: special-config
          key: special.type
    ### 引用方式2
    envFrom:
    - configMapRef:
        name: game-config-file   ### 这个name的值来自 ConfigMap
  restartPolicy: Never
# 查看Pod
[root@k8s-master volume]# kubectl get po pod-configmap-env
NAME                READY   STATUS      RESTARTS   AGE
pod-configmap-env   0/1     Completed   0          21s
# 查看Pod日志
[root@k8s-master volume]# kubectl logs pod-configmap-env 
MYAPP_SVC_PORT_80_TCP_ADDR=10.98.57.156
KUBERNETES_PORT=tcp://192.168.0.1:443
KUBERNETES_SERVICE_PORT=443
MYAPP_SVC_PORT_80_TCP_PORT=80
HOSTNAME=pod-configmap-env
SHLVL=1
MYAPP_SVC_PORT_80_TCP_PROTO=tcp
HOME=/root
####出自ConfigMap
SPECAIL_HOW_KEY=very
game.properties=enemies=aliens
lives=3
enemies.cheat=true
enemies.cheat.level=noGoodRotten
secret.code.passphrase=UUDDLRLRBABAs
secret.code.allowed=true
secret.code.lives=30

SPECAIL_TPYE_KEY=charm
####出自ConfigMap
MYAPP_SVC_PORT_80_TCP=tcp://10.98.57.156:80
...
```

## 3. 使用ConfigMap设置命令行参数

```shell
# 创建yaml文件
[root@k8s-master volume]# vi pod_configmap_cmd.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-configmap-cmd
spec:
  containers:
  - name: myapp
    image: registry.cn-beijing.aliyuncs.com/google_registry/myapp:v1
    command: ["/bin/sh", "-c", "echo \"===$(SPECAIL_HOW_KEY)===$(SPECAIL_TPYE_KEY)===\""]
    env:
    - name: SPECAIL_HOW_KEY
      valueFrom:
        configMapKeyRef:
          name: special-config
          key: special.how
    - name: SPECAIL_TPYE_KEY
      valueFrom:
        configMapKeyRef:
          name: special-config
          key: special.type
  restartPolicy: Never
# 创建Pod并查看日志
[root@k8s-master volume]# kubectl apply -f pod_configmap_cmd.yaml 
pod/pod-configmap-cmd created
[root@k8s-master volume]# kubectl get po pod-configmap-cmd
NAME                READY   STATUS      RESTARTS   AGE
pod-configmap-cmd   0/1     Completed   0          51s
[root@k8s-master volume]# kubectl logs pod-configmap-cmd
===very===charm===
```

## 4. 通过数据卷插件使用ConfigMap

在数据卷里面使用ConfigMap,最基本的就是将文件填入数据卷,在这个文件中,键就是文件名【第一层级的键】,键值就是文件内容.

```shell
[root@k8s-master volume]# vi pod_configmap_volume.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-configmap-volume
spec:
  containers:
  - name: myapp
    image: registry.cn-beijing.aliyuncs.com/google_registry/myapp:v1
    command: ["/bin/sh", "-c", "sleep 600"]
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: configmap-demo
  restartPolicy: Never
[root@k8s-master volume]# kubectl apply -f pod_configmap_volume.yaml 
pod/pod-configmap-volume created
[root@k8s-master volume]# kubectl get po pod-configmap-volume
NAME                   READY   STATUS    RESTARTS   AGE
pod-configmap-volume   1/1     Running   0          13s
# 先查看一下configmap-demo
[root@k8s-master volume]# kubectl describe cm configmap-demo
Name:         configmap-demo
Namespace:    default
Labels:       <none>
Annotations:  <none>
# 注意这里的四个数据段
Data
====
game.properties:
----
enemy.types=aliens,monsters
player.maximum-lives=5

player_initial_lives:
----
3
ui_properties_file_name:
----
user-interface.properties
user-interface.properties:
----
color.good=green
color.bad=yellow

Events:  <none>
# 进入pod内查看配置
[root@k8s-master volume]# kubectl exec -it pod-configmap-volume sh
/ # cd /etc/config/
/etc/config # ls
game.properties            player_initial_lives       ui_properties_file_name    user-interface.properties
/etc/config # cat game.properties 
enemy.types=aliens,monsters
player.maximum-lives=5
/etc/config # 
/etc/config # cat player_initial_lives 
3/etc/config # 
/etc/config # cat ui_properties_file_name 
user-interface.properties/etc/config # 
/etc/config # cat user-interface.properties 
color.good=green
color.bad=yellow
/etc/config # 
```

## 5. ConfigMap热更新

### 5.1. 准备工作

```shell
# 创建yaml文件,创建ConfigMap和Deploy
[root@k8s-master volume]# vi pod_configmap_hot.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: log-config
  namespace: default
data:
  log_level: INFO
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deploy
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
      release: v1
  template:
    metadata:
      labels:
        app: myapp
        release: v1
        env: test
    spec:
      containers:
      - name: myapp
        image: registry.cn-beijing.aliyuncs.com/google_registry/myapp:v1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
        volumeMounts:
        - name: config-volume
          mountPath: /etc/config
      volumes:
      - name: config-volume
        configMap:
          name: log-config
# 创建并查看
[root@k8s-master volume]# kubectl apply -f pod_configmap_hot.yaml 
configmap/log-config created
deployment.apps/myapp-deploy created
[root@k8s-master volume]# kubectl get configmap log-config
NAME         DATA   AGE
log-config   1      12s
[root@k8s-master volume]# kubectl get po -o wide
NAME                            READY   STATUS             RESTARTS   AGE   IP               NODE         NOMINATED NODE   READINESS GATES
myapp-deploy-6f754c5944-748x2   1/1     Running            0          85s   172.16.36.96     k8s-node1    <none>           <none>
myapp-deploy-6f754c5944-qh7bk   1/1     Running            0          85s   172.16.36.98     k8s-node1    <none>           <none>
[root@k8s-master volume]# kubectl describe cm log-config
Name:         log-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
log_level:
----
INFO
Events:  <none>
# 查看deploy中的log_level
[root@k8s-master volume]# kubectl exec -it myapp-deploy-6f754c5944-748x2 -- cat /etc/config/log_level
INFO
```

### 5.2. 热更新log_level

```shell
# 修改log_level=DEBUG
[root@k8s-master volume]# kubectl edit cm log-config
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
data:
  log_level: DEBUG
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"log_level":"INFO"},"kind":"ConfigMap","metadata":{"annotations":{},"name":"log-config","namespace":"default"}}
  creationTimestamp: "2020-12-15T02:48:09Z"
  name: log-config
  namespace: default
  resourceVersion: "336806"
  selfLink: /api/v1/namespaces/default/configmaps/log-config
  uid: acbbe6ce-d177-422a-8c08-e3f3a77d4ebb
# 查看对应的configmap, log_level已修改为DEBUG
[root@k8s-master volume]# kubectl describe cm log-config
Name:         log-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
log_level:
----
DEBUG
Events:  <none>
# 查看pod中log_level已修改为DEBUG
[root@k8s-master volume]# kubectl exec -it myapp-deploy-6f754c5944-748x2 -- cat /etc/config/log_level
DEBUG
```
