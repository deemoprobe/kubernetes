# Kubernetes基础之Pod

Pod是Kubernetes中创建管理和可部署的最小计算单元，它由一个或多个容器组成，每个Pod还包含了一个Pause容器，Pause容器是Pod的父容器，主要负责僵尸进程的回收管理，通过Pause容器可以使同一个Pod里面的多个容器共享存储、网络、PID、IPC等。

## 使用Pod

Pod 中的容器被自动安排到集群中的同一物理机或虚拟机上，并可以一起进行调度。通常情况下，很少去直接创建Pod单实例，一般是通过Pod控制器（ReplicaSet、Deployment、DaemonSet等）进行创建和管理，并且这些高级资源具有Pod所不具备的很多高级特性（比如：自由伸缩，回滚等）。控制器创建Pod后将Pod调度到集群的节点上，Pod会保持在该节点上运行，直到Pod结束执行、Pod对象被删除、Pod因资源不足而被驱逐或者节点失效为止。

> 说明： 重启 Pod 中的容器不应与重启 Pod 混淆。 Pod 不是进程，而是容器运行的环境。 在被删除之前，Pod 会一直存在。

- **Pod**yaml文件示例

```bash
apiVersion: v1       # 必选，API的版本号
kind: Pod            # 必选，类型Pod
metadata:            # 必选，元数据
  name: nginx        # 必选，符合RFC1035规范的Pod名称
  namespace: default # 可选，Pod所在的命名空间，默认为default
  labels:            # 可选，标签选择器，一般用于过滤和区分Pod
    app: nginx
    role: frontend   # 可以写多个
  annotations:       # 可选，注释列表，可以写多个
    app: nginx
spec:                # 必选，用于定义容器的详细信息
  initContainers:    # 初始化容器，在容器启动之前执行的一些初始化操作
  - command:
    - sh
    - -c
    - echo "I am InitContainer for init some configuration"
    image: busybox
    imagePullPolicy: IfNotPresent
    name: init-container
  containers:                # 必选，容器列表
  - name: nginx              # 必选，符合RFC 1035规范的容器名称
    image: nginx:latest      # 必选，容器所用的镜像的地址
    imagePullPolicy: Always  # 可选，镜像拉取策略
    command:                 # 可选，容器启动执行的命令
    - nginx 
    - -g
    - "daemon off;"
    workingDir: /usr/share/nginx/html  # 可选，容器的工作目录
    volumeMounts:                      # 可选，存储卷配置，可以配置多个
    - name: webroot                    # 存储卷名称
      mountPath: /usr/share/nginx/html # 挂载目录
      readOnly: true                   # 只读
    ports:                 # 可选，容器需要暴露的端口号列表
    - name: http           # 端口名称
      containerPort: 80    # 端口号
      protocol: TCP        # 端口协议，默认TCP
    env:                   # 可选，环境变量配置列表
    - name: TZ             # 变量名
      value: Asia/Shanghai # 变量的值
    - name: LANG
      value: en_US.utf8
    resources:      # 可选，资源限制和资源请求限制
      limits:       # 最大限制设置
        cpu: 1000m
        memory: 1024Mi
      requests:     # 启动所需的资源
        cpu: 100m
        memory: 512Mi
#    startupProbe:  # 可选，检测容器内进程是否完成启动。注意三种检查方式同时只能使用一种。
#      httpGet:     # httpGet检测方式，生产环境建议使用httpGet实现接口级健康检查，健康检查由应用程序提供。
#            path: /api/successStart # 检查路径
#            port: 80
    readinessProbe: # 可选，健康检查。注意三种检查方式同时只能使用一种。
      httpGet:      # httpGet检测方式，生产环境建议使用httpGet实现接口级健康检查，健康检查由应用程序提供。
            path: / # 检查路径
            port: 80 # 监控端口
    #livenessProbe: # 可选，健康检查
      #exec:        # 执行容器命令检测方式
            #command: 
            #- cat
            #- /health
    #httpGet:       # httpGet检测方式
    #   path: /_health # 检查路径
    #   port: 8080
    #   httpHeaders: # 检查的请求头
    #   - name: end-user
    #     value: Jason 
      tcpSocket:     # 端口检测方式
            port: 80
      initialDelaySeconds: 60  # 初始化时间
      timeoutSeconds: 2        # 超时时间
      periodSeconds: 20        # 检测间隔
      successThreshold: 1      # 检查成功为2次表示就绪
      failureThreshold: 2      # 检测失败1次表示未就绪
    lifecycle:
      postStart: # 容器创建完成后执行的指令, 可以是exec httpGet TCPSocket
        exec:
          command:
          - sh
          - -c
          - 'mkdir /data/'
      preStop:
        httpGet:      
              path: /
              port: 80
      #  exec:
      #    command:
      #    - sh
      #    - -c
      #    - sleep 9
  restartPolicy: Always   # 可选，默认为Always
  #nodeSelector: # 可选，指定Node节点
  #      region: subnet7
  imagePullSecrets:  # 可选，拉取镜像使用的secret，可以配置多个
  - name: default-dockercfg-86258
  hostNetwork: false  # 可选，是否为主机模式，如是，会占用主机端口
  volumes:            # 共享存储卷列表
  - name: webroot     # 名称，与上面volumeMounts对应
    emptyDir: {}      # 挂载目录
        #hostPath:    # 挂载本机目录
        #  path: /etc/hosts
```

> 在高级资源中创建Pod是通过文件中`template` Pod模板字段来创建Pod

```bash
# 例如创建Job，
apiVersion: batch/v1
kind: Job
metadata:
  name: hello
spec:
  template:
    # 这里是 Pod 模版
    spec:
      containers:
      - name: hello
        image: busybox
        command: ['sh', '-c', 'echo "Hello, Kubernetes!" && sleep 3600']
      restartPolicy: OnFailure
    # 以上为 Pod 模版
```

## Pod探针

- StartupProbe（1.16版本后）：用于判断容器内应用程序是否已经启动。如果配置了startupProbe，会先禁止其他的探测，直到它成功为止（直到容器程序成功启动为止），成功后会进入其他探针的探测。
- LivenessProbe：用于探测容器是否运行，如果探测失败，kubelet会根据配置的重启策略进行相应的处理。若没有配置该探针，默认就是success。
- ReadinessProbe：一般用于探测容器内的程序是否就绪，它的返回值如果为success，那么久代表这个容器已经完成启动，并且程序已经是可以接受流量的状态。

## Pod探针的检测方式

- ExecAction：在容器内执行一个命令，如果返回值为0，则认为容器健康。
- TCPSocketAction：通过TCP连接检查容器内的端口是否是通的，如果是通的就认为容器健康。
- HTTPGetAction：通过应用程序暴露的API地址来检查程序是否是正常的，如果状态码为200~400之间，则认为容器健康。

> 下面案例coredns的健康检查配置中`LivenessProbe`通过`httpGet`的方式检测提供`8080/health`接口，检查容器是否处于运行状态，检查通过后接着进行`ReadinessProbe`探针检测，`httpGet`方式检测提供的`8181/ready`接口，检测通过则表示容器内程序运行正常，可以接收流量。健康检查接口需要在镜像中预定程序源码中写好，为后续接入健康检查做准备。

```bash
# 以coredns为例，可以看到健康检查配置了LivenessProbe和ReadinessProbe
[root@k8s-master01 ~]# kubectl get deploy -n kube-system coredns -oyaml > coredns.yaml
...
    spec:
      containers:
        ...
        livenessProbe:
          failureThreshold: 5
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 5
        ...
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /ready
            port: 8181
            scheme: HTTP
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 11
          ...
```

> **特殊情况**：如果遇到容器内程序启动时间太长，超过了`initialDelaySeconds`设定的容器初始化时间，那么健康检查也不会通过，容器会不断被杀掉重启，造成服务无法正常运行。像上面这些情况就需要检测容器内程序是否健康，此时可以使用`startupProbe`。

## 探针检查参数配置

```bash
initialDelaySeconds: 10   # 容器启动后且依旧存活，等待10s后开始检测。若不配置，默认为0s
timeoutSeconds: 2         # 检测超时后的等待时间，默认为1s
periodSeconds: 20         # 检测间隔时间，默认为10s
successThreshold: 1       # 检测连续成功1次后标记Pod就绪
failureThreshold: 2       # 检测连续失败2次后标记Pod失败
```

## StartupProbe实例

采用HTTPGetAction探测方式

> 实例使用的nginx:latest镜像并没有预设下面所写的健康检查接口，所以检测是不会通过的，可以通过`Events`查看。重启策略如果不指定，默认是`restartPolicy: Always`，失败后会一直重启

```bash
[root@k8s-master01 ~]# vim startupprobe-httpget.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: startupprobe-httpget
  name: startupprobe-httpget
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 8080
    startupProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 15
      timeoutSeconds: 1
[root@k8s-master01 ~]# kubectl apply -f startupprobe-httpget.yaml 
pod/startupprobe-httpget created
[root@k8s-master01 ~]# kubectl describe po startupprobe-httpget 
...
    Startup:        http-get http://:8080/health delay=15s timeout=1s period=10s #success=1 #failure=3
...
Events:
  Type     Reason     Age               From               Message
  ----     ------     ----              ----               -------
  Normal   Scheduled  56s               default-scheduler  Successfully assigned default/startupprobe-httpget to k8s-node02
  Normal   Pulling    56s               kubelet            Pulling image "nginx"
  Normal   Pulled     40s               kubelet            Successfully pulled image "nginx" in 15.591936231s
  Normal   Created    40s               kubelet            Created container nginx
  Normal   Started    40s               kubelet            Started container nginx
  Warning  Unhealthy  7s (x2 over 17s)  kubelet            Startup probe failed: Get "http://172.27.14.202:8080/health": dial tcp 172.27.14.202:8080: connect: connection refused
# 查看Pod状态，健康检查不通过，处于Running但并没有Ready，并且还在不断重启
[root@k8s-master01 ~]# kubectl get po
NAME                   READY   STATUS    RESTARTS      AGE
startupprobe-httpget   0/1     Running   5 (20s ago)   4m40s
```

## LivenessProbe实例

采用ExecAction探测方式

```bash
# 创建liveness.yaml
[root@k8s-master01 ~]# vim livenessprobe-exec.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: livenessprobe-exec
  name: livenessprobe-exec
spec:
  containers:
  - name: liveness
    image: busybox
    args:
    - /bin/sh
    - -c
    - echo ok > /tmp/healthy; sleep 10; rm -rf /tmp/healthy; sleep 60
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 15
      periodSeconds: 5
[root@k8s-master01 ~]# kubectl apply -f livenessprobe-exec.yaml
pod/livenessprobe-exec created
[root@k8s-master01 ~]# kubectl describe po livenessprobe-exec
...
    Liveness:       exec [cat /tmp/healthy] delay=15s timeout=1s period=5s #success=1 #failure=3
...
Events:
  Type     Reason     Age   From               Message
  ----     ------     ----  ----               -------
  Normal   Scheduled  34s   default-scheduler  Successfully assigned default/livenessprobe-exec to k8s-node02
  Normal   Pulling    33s   kubelet            Pulling image "busybox"
  Normal   Pulled     20s   kubelet            Successfully pulled image "busybox" in 13.224110856s
  Normal   Created    20s   kubelet            Created container liveness
  Normal   Started    20s   kubelet            Started container liveness
  Warning  Unhealthy  4s    kubelet            Liveness probe failed: cat: can not open '/tmp/healthy': No such file or directory
# 观察Pod状态可以发现一开始是正常的，后面因为健康检查失败重启了
[root@k8s-master01 ~]# kubectl get po
NAME                 READY   STATUS    RESTARTS   AGE
livenessprobe-exec   1/1     Running   0          75s
[root@k8s-master01 ~]# kubectl get po
NAME                 READY   STATUS    RESTARTS      AGE
livenessprobe-exec   1/1     Running   1 (21s ago)   91s
```

> 在这个配置文件中，Pod 中只有一个容器。`periodSeconds`字段指定了 kubelet 应该每 5 秒执行一次存活探测。`initialDelaySeconds`字段告诉 kubelet 在执行第一次探测前应该等待 15 秒。 kubelet 在容器内执行命令`cat /tmp/healthy`来进行探测。 如果命令执行成功并且返回值为 0，kubelet 就会认为这个容器是健康存活的。 如果这个命令返回非 0 值，kubelet 会杀死这个容器并重新启动它。

## ReadinessProbe实例

采用TCPSocketAction探测方式

> nginx启动后默认开启80端口，下面健康检查会通过

```bash
[root@k8s-master01 ~]# vim readinessprobe-tcpsocket.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: readinessprobe-tcpsocket
  name: readinessprobe-tcpsocket
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
    ports:
    - containerPort: 80
    readinessProbe:
      tcpSocket:
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 10
[root@k8s-master01 ~]# kubectl apply -f readinessprobe-tcpsocket.yaml 
pod/readinessprobe-tcpsocket created
# 可以看到Pod变为Ready状态
[root@k8s-master01 ~]# kubectl get po readinessprobe-tcpsocket 
NAME                       READY   STATUS    RESTARTS   AGE
readinessprobe-tcpsocket   0/1     Running   0          19s
[root@k8s-master01 ~]# kubectl get po readinessprobe-tcpsocket 
NAME                       READY   STATUS    RESTARTS   AGE
readinessprobe-tcpsocket   1/1     Running   0          20s
[root@k8s-master01 ~]# kubectl describe po readinessprobe-tcpsocket 
...
    Readiness:      tcp-socket :80 delay=5s timeout=1s period=10s #success=1 #failure=3
...
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  112s  default-scheduler  Successfully assigned default/readinessprobe-tcpsocket to k8s-node02
  Normal  Pulling    112s  kubelet            Pulling image "nginx"
  Normal  Pulled     102s  kubelet            Successfully pulled image "nginx" in 9.699109421s
  Normal  Created    102s  kubelet            Created container nginx
  Normal  Started    102s  kubelet            Started container nginx
```

> `Successfully pulled image "nginx" in 9.699109421s`拉取镜像花时间约9.7s，健康检查间隔时间是10s，所以大约在19.7sPod标记为Running状态

## Pod生命周期

![kubernetes-pod-life-cycle](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/kubernetes-pod-life-cycle.jpg)

总结生命周期

## Pod退出流程

用户删除 Pod。
Pod 进入 Terminating 状态。
与此同时，K8s 会将 Pod 从对应的 service 上摘除。
与此同时，针对有 PreStop Hook 的容器，kubelet 会调用每个容器的 PreStop Hook，假如 PreStop Hook 的运行时间超出了 grace period，kubelet 会发送 SIGTERM 并再等 2 秒。
与此同时，针对没有 PreStop Hook 的容器，kubelet 发送 SIGTERM。
grace period 超出之后，kubelet 发送 SIGKILL 干掉尚未退出的容器。

<https://freecontent.manning.com/handling-client-requests-properly-with-kubernetes/>

<https://insights.thoughtworks.cn/how-to-stop-kubernetes-elegantly/>

总结优雅退出流程
