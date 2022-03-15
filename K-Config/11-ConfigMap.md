# Kubernetes配置管理之ConfigMap

2021-0720

## 概念

ConfigMap是一种API对象，用来将非机密性的数据保存到健值对中。使用时可以用作环境变量、命令参数或者存储卷中的配置文件。提供向容器内注入信息的功能。

ConfigMap将环境配置信息和容器镜像解耦，便于应用配置的修改。

备注: ConfigMap 并不提供保密或者加密功能。如果想存储的数据是机密的，可以使用Secret或其他第三方工具来保证数据的私密性。而不是用 ConfigMap。

ConfigMap的一些**限制**:

- 如需获取ConfigMap的内容到容器，则ConfigMap必须在Pod之前创建
- ConfigMap受Namespace限制，Pod只有和ConfigMap处于相同Namespace下的才可以引用它
- 静态Pod无法引用ConfigMap
- 使用volumeMounts给Pod挂载ConfigMap时，mountPath只能是目录，而不能是文件；容器内部该目录下如果原有文件，那么将会被新挂载的ConfigMap覆盖

## 使用ConfigMap

### 通过目录创建

`--from-file`指定目录下所有文件都作为ConfigMap中的键值对来使用，键为文件名，值为文件内容。

```bash
[root@k8s-master01 ~]# mkdir -p yamls/configmap
[root@k8s-master01 ~]# cd yamls/configmap/
[root@k8s-master01 configmap]# vim game.properties
enemies=aliens
lives=3
enemies.cheat=true
enemies.cheat.level=noGoodRotten
secret.code.passphrase=UUDDLRLRBABAs
secret.code.allowed=true
secret.code.lives=30
[root@k8s-master01 configmap]# vim ui.properties
color.good=purple
color.bad=yellow
allow.textmode=true
how.nice.to.look=fairlyNice
[root@k8s-master01 configmap]# kubectl create configmap game --from-file=/root/yamls/configmap/
configmap/game created
[root@k8s-master01 configmap]# kubectl get cm
NAME               DATA   AGE
game               2      44s
[root@k8s-master01 configmap]# kubectl get cm game -oyaml
apiVersion: v1
data:
  game.properties: | # |后为文件键值对内容
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
  creationTimestamp: "2022-03-15T01:27:41Z"
  name: game
  namespace: default
  resourceVersion: "161625"
  uid: 784d08a8-1d6e-4f5f-ab2e-330348d4bf69
# 也可以通过describe命令查看
[root@k8s-master01 configmap]# kubectl describe cm game 
Name:         game
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

ui.properties:
----
color.good=purple
color.bad=yellow
allow.textmode=true
how.nice.to.look=fairlyNice


BinaryData # 用来保存二进制数据作为base64编码字串
====

Events:  <none>
```

### 通过文件创建

`--from-file`指定对应配置文件, 可以多次使用引入不同的配置文件

```bash
[root@k8s-master01 configmap]# kubectl create configmap game-ui --from-file=/root/yamls/configmap/ui.properties 
configmap/game-ui created
[root@k8s-master01 configmap]# kubectl get cm
NAME               DATA   AGE
game               2      6m44s
game-ui            1      14s
[root@k8s-master01 configmap]# kubectl describe cm game-ui 
Name:         game-ui
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
ui.properties:
----
color.good=purple
color.bad=yellow
allow.textmode=true
how.nice.to.look=fairlyNice


BinaryData
====

Events:  <none>
```

### 通过命令行创建

`--from-literal`传递配置参数, 可多次使用

```bash
[root@k8s-master01 configmap]# kubectl create configmap game-cli --from-literal=game.name=gogogo --from-literal="game.date=2022-03"
configmap/game-cli created
[root@k8s-master01 configmap]# kubectl describe cm game-cli 
Name:         game-cli
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
game.date:
----
2022-03
game.name:
----
gogogo

BinaryData
====

Events:  <none>
```

### 通过YAML文件创建

```bash
[root@k8s-master01 configmap]# vim game-yaml.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-yaml
data:
  # 类属性键;每一个键都映射到一个简单的值
  player_initial_lives: "3"
  ui_properties_file_name: 'user-interface.properties'
  # 类文件键
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5
  user-interface.properties: |
    color.good=green
    color.bad=yellow
[root@k8s-master01 configmap]# kubectl apply -f game-yaml.yaml 
configmap/game-yaml created
[root@k8s-master01 configmap]# kubectl describe cm game-yaml
Name:         game-yaml
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


BinaryData
====

Events:  <none>
```

## Pod中使用ConfigMap

在Pod中加载ConfigMap配置

### 查看当前的configmap

```bash
[root@k8s-master01 configmap]# kubectl get cm
NAME               DATA   AGE
game               2      16m
game-cli           2      5m13s
game-ui            1      10m
game-yaml          4      2m32s
```

### 作为环境变量引用

```bash
[root@k8s-master01 configmap]# vim pod-from-cm.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-from-cm
spec:
  containers:
  - name: myapp
    image: registry.cn-beijing.aliyuncs.com/google_registry/myapp:v1
    command: ["/bin/sh", "-c", "env"]
    ### 引用方式1，直接应用键值对
    env:
    - name: GAME_NAME # 指定环境变量名
      valueFrom:
        configMapKeyRef:
          name: game-cli   ### 这个name的值来自 ConfigMap
          key: game.name       ### 这个key的值为需要取值的键
    - name: GAME_DATE
      valueFrom:
        configMapKeyRef:
          name: game-cli
          key: game.date
    ### 引用方式2，引用ConfigMap文件
    envFrom:
    - configMapRef:
        name: game   ### 这个name的值来自 ConfigMap
  restartPolicy: Never
[root@k8s-master01 configmap]# kubectl apply -f pod-from-cm.yaml 
pod/pod-from-cm created
[root@k8s-master01 configmap]# kubectl get po pod-from-cm 
NAME          READY   STATUS      RESTARTS   AGE
pod-from-cm   0/1     Completed   0          5s
# 查看Pod日志，可见相关环境变量已经导入
[root@k8s-master01 configmap]# kubectl logs pod-from-cm 
KUBERNETES_SERVICE_PORT=443
MYAPP_SVC_PORT_80_TCP_ADDR=10.98.57.156
KUBERNETES_PORT=tcp://10.96.0.1:443
ui.properties=color.good=purple
color.bad=yellow
allow.textmode=true
how.nice.to.look=fairlyNice

MYAPP_SVC_PORT_80_TCP_PORT=80
HOSTNAME=pod-from-cm
SHLVL=1
GAME_DATE=2022-03
MYAPP_SVC_PORT_80_TCP_PROTO=tcp
HOME=/root
NGINX_PORT_80_TCP=tcp://10.100.71.223:80
game.properties=enemies=aliens
lives=3
enemies.cheat=true
enemies.cheat.level=noGoodRotten
secret.code.passphrase=UUDDLRLRBABAs
secret.code.allowed=true
secret.code.lives=30

GAME_NAME=gogogo
...
```

## 输出环境变量

```bash
# 创建yaml文件
[root@k8s-master01 configmap]# vim pod-cm-env.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: pod-cm-env
spec:
  containers:
  - name: myapp
    image: registry.cn-beijing.aliyuncs.com/google_registry/myapp:v1
    command: ["/bin/sh", "-c", "echo \"Game name is $(GAME_NAME_AGAIN)\nGame date is $(GAME_DATE_AGAIN)\""]
    env:
    - name: GAME_NAME_AGAIN
      valueFrom:
        configMapKeyRef:
          name: game-cli
          key: game.name
    - name: GAME_DATE_AGAIN
      valueFrom:
        configMapKeyRef:
          name: game-cli
          key: game.date
  restartPolicy: Never
[root@k8s-master01 configmap]# kubectl apply -f pod-cm-env.yaml 
pod/pod-cm-env created
[root@k8s-master01 configmap]# kubectl get po pod-cm-env 
NAME         READY   STATUS      RESTARTS   AGE
pod-cm-env   0/1     Completed   0          6s
[root@k8s-master01 configmap]# kubectl logs pod-cm-env 
Game name is gogogo
Game date is 2022-03
```

## 数据卷中使用

```bash
[root@k8s-master01 configmap]# vim po-cm-vol.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-cm-vol
spec:
  containers:
  - name: myapp
    image: registry.cn-beijing.aliyuncs.com/google_registry/myapp:v1
    command: ["/bin/sh", "-c", "sleep 600"] # 延迟十分钟再进入completed
    volumeMounts:
    - name: config-volume
      mountPath: "/config"
      readOnly: true
  volumes:
  - name: config-volume
    configMap:
      name: game
      items: # 如果不指定，则调用该cm下所有配置
      - key: "game.properties"
        path: "game.properties"
  restartPolicy: Never
[root@k8s-master01 configmap]# kubectl apply -f pod-cm-vol.yaml 
pod/pod-cm-vol created
[root@k8s-master01 configmap]# kubectl get po pod-cm-vol 
NAME         READY   STATUS    RESTARTS   AGE
pod-cm-vol   1/1     Running   0          8s
# 确认cm配置
[root@k8s-master01 configmap]# kubectl describe cm game
Name:         game
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

ui.properties:
----
color.good=purple
color.bad=yellow
allow.textmode=true
how.nice.to.look=fairlyNice


BinaryData
====

Events:  <none>
# 进入pod-cm-vol内查看配置挂载的文件
[root@k8s-master volume]# kubectl exec -it pod-configmap-volume sh
[root@k8s-master01 configmap]# kubectl exec -it pod-cm-vol -- sh
/ # ls /config
game.properties
/ # cat /config/game.properties # 此文件链接的文件为只读文件，通常是真实数据文件的软链接
enemies=aliens
lives=3
enemies.cheat=true
enemies.cheat.level=noGoodRotten
secret.code.passphrase=UUDDLRLRBABAs
secret.code.allowed=true
secret.code.lives=30
```

## ConfigMap热更新

### 准备工作

```bash
# 创建yaml文件,创建ConfigMap和Deploy
[root@k8s-master01 configmap]# vim pod-cm-hot.yaml
      app: myapp
    matchLabels:
  selector:
  replicas: 2
        release: v1
apiVersion: v1
kind: ConfigMap
metadata:
  name: cm-loglevel
  namespace: default
data:
  log_level: INFO
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mydeploy
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
    spec:
      containers:
      - name: myapp
        image: registry.cn-beijing.aliyuncs.com/google_registry/myapp:v1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
        volumeMounts:
        - name: config-volume
          mountPath: "/config"
      volumes:
      - name: config-volume
        configMap:
          name: cm-loglevel
[root@k8s-master01 configmap]# kubectl apply -f pod-cm-hot.yaml 
configmap/cm-loglevel created
deployment.apps/mydeploy created
[root@k8s-master01 configmap]# kubectl get deployments.apps mydeploy 
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
mydeploy   2/2     2            2           16s
[root@k8s-master01 configmap]# kubectl get cm cm-loglevel 
NAME          DATA   AGE
cm-loglevel   1      28s
[root@k8s-master01 configmap]# kubectl describe cm cm-loglevel 
Name:         cm-loglevel
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
log_level:
----
INFO

BinaryData
====

Events:  <none>
[root@k8s-master01 configmap]# kubectl get po -owide | grep mydeploy
mydeploy-77c886996d-wbnrs   1/1     Running     0             100s    172.18.195.14    k8s-master03   <none>           <none>
mydeploy-77c886996d-zrg8n   1/1     Running     0             101s    172.27.14.204    k8s-node02     <none>           <none>
# 查看pod中的log_level
[root@k8s-master01 configmap]# kubectl exec -it mydeploy-77c886996d-wbnrs -- cat /config/log_level
INFO
```

### 热更新log_level

```bash
# 修改log_level=DEBUG
[root@k8s-master01 configmap]# kubectl edit cm cm-loglevel 
apiVersion: v1
data:
  log_level: DEBUG
kind: ConfigMap
...
# 查看对应的configmap, log_level已修改为DEBUG
[root@k8s-master01 configmap]# kubectl describe cm cm-loglevel 
Name:         cm-loglevel
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
log_level:
----
DEBUG

BinaryData
====

Events:  <none>
# 查看pod中log_level已修改为DEBUG
[root@k8s-master01 configmap]# kubectl exec -it mydeploy-77c886996d-wbnrs -- cat /config/log_level
DEBUG
```

## 不可变的ConfigMap

为了确保ConfigMap不会被无意中修改，可以设置不可变的ConfigMap。此功能特性由`ImmutableEphemeralVolumes`控制。可以通过将`immutable`字段设置为`true`创建不可变更的ConfigMap。一旦某ConfigMap被标记为不可变更，则无法逆转这一变化，也无法更改data或binaryData字段的内容。只能删除并重建 ConfigMap。

```bash
[root@k8s-master01 configmap]# vim cm-immutable.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cm-immutable
data:
  name: deemoprobe
  date: 2022-03
  label: test
immutable: true
[root@k8s-master01 configmap]# kubectl apply -f cm-immutable.yaml 
configmap/cm-immutable created
[root@k8s-master01 configmap]# kubectl describe cm cm-immutable 
Name:         cm-immutable
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
name:
----
deemoprobe
date:
----
2022-03
label:
----
test

BinaryData
====

Events:  <none>
# 尝试修改yaml文件应用时会提示如下
[root@k8s-master01 configmap]# kubectl apply -f cm-immutable.yaml 
The ConfigMap "cm-immutable" is invalid: data: Forbidden: field is immutable when `immutable` is set
# 尝试使用edit命令修改会提示如下
[root@k8s-master01 configmap]# kubectl edit cm cm-immutable
# configmaps "cm-immutable" was not valid:
# * data: Forbidden: field is immutable when `immutable` is set
```
