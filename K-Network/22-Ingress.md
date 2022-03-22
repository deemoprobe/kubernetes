# Kubernetes网络之Ingress

2021-0816

## 概念

> 本文通过Helm部署Nginx Ingress Controller

- `Ingress`是集群外部访问Kubernetes的一个入口，将外部的请求转发到集群内不同的 Service 上，相当于 nginx、haproxy 等负载均衡器
- `Ingress Controller`可以理解为一个监听器，通过不断地监听 kube-apiserver，实时的感知后端 Service、Pod 的变化，当得到这些信息变化后，Ingress Controller 再结合 Ingress 的配置，更新反向代理负载均衡器，达到服务发现的作用。
- `NGINX Ingress Controller`是使用 Kubernetes Ingress 资源对象构建的，用 ConfigMap 来存储 Nginx 配置的一种 Ingress Controller 实现。要使用 Ingress 对外暴露服务，就需要提前安装一个 Ingress Controller，常见的就是`NGINX Ingress Controller`，生产环境一般需要部署多个`ingress-nginx-controller`实例实现高可用（不要部署在master节点，至少部署在3个独立的Node节点）。

> 官方列举出了多种`Ingress Controller`：<https://kubernetes.io/zh/docs/concepts/services-networking/ingress-controllers/>

## Ingress图解

Ingress 公开了从集群外部到集群内 services 的 HTTP 和 HTTPS 路由. 流量路由由 Ingress 资源上定义的规则控制.

![20210112135036](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20210112135036.png)

可以将 Ingress 配置为提供服务外部可访问的 URL、负载均衡流量、 SSL / TLS,以及提供基于名称的虚拟主机.Ingress 控制器 通常负责通过负载均衡器来实现 Ingress,尽管它也可以配置边缘路由器或其他前端来帮助处理流量.

Ingress 不会公开任意端口或协议.若将 HTTP 和 HTTPS 以外的服务公开到 Internet 时,通常使用 Service.Type=NodePort 或者 Service.Type=LoadBalancer 类型的服务.

本实验Nginx Ingress架构示意图:

![Ingress-Nginx](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/Ingress-Nginx.jpg)

## 部署Helm3.x

helm通过打包的方式,支持发布的版本管理和控制,很大程度上简化了Kubernetes应用的部署和管理.

Helm本质就是让k8s的应用管理(Deployment、Service等)可配置,能动态生成.通过动态生成K8S资源清单文件(deployment.yaml、service.yaml).然后kubectl自动调用K8S资源部署.

> 说明: Helm3.x 版本已经不需要再安装tiller(之前老版本中的Helm仓库的服务端), 直接安装配置好仓库就可以使用了

```shell
# 在官方(https://github.com/helm/helm/releases)下载想要的的版本, 当前(2022-01-21)最新稳定版 V3.7.2
# 解压并配置
[root@k8s-master01 ~]# tar -zxvf helm-v3.7.2-linux-amd64.tar.gz 
linux-amd64/
linux-amd64/helm
linux-amd64/LICENSE
linux-amd64/README.md
[root@k8s-master01 ~]# mv linux-amd64/helm /usr/local/bin/helm
[root@k8s-master01 ~]# helm version
version.BuildInfo{Version:"v3.7.2", GitCommit:"663a896f4a815053445eec4153677ddc24a0a361", GitTreeState:"clean", GoVersion:"go1.16.10"}
# 添加仓库
[root@k8s-master01 ingress]# helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
"ingress-nginx" has been added to your repositories
[root@k8s-master01 ingress]# helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "ingress-nginx" chart repository
Update Complete. ⎈Happy Helming!⎈
[root@k8s-master01 ingress]# helm repo list
NAME            URL                                       
ingress-nginx   https://kubernetes.github.io/ingress-nginx
```

## Helm部署 Nginx-Ingress

```shell
# 查看nginx-ingress资源包，显示为最新版
[root@k8s-master01 ingress]# helm search repo ingress-nginx
NAME                            CHART VERSION   APP VERSION     DESCRIPTION                                       
ingress-nginx/ingress-nginx     4.0.16          1.1.1           Ingress controller for Kubernetes using NGINX a...
# 1.获取最新版本压缩包
[root@k8s-master01 ingress]# helm pull ingress-nginx/ingress-nginx
# 2.获取最新版本并解压，不保留压缩包
[root@k8s-master01 ingress]# helm pull ingress-nginx/ingress-nginx --untar
# 3.获取指定版本并解压，不保留压缩包
[root@k8s-master01 ingress]# helm pull ingress-nginx/ingress-nginx --version=4.0.16 --untar
# 本实验采用第3种方式获取了ingress-nginx-4.0.16.tgz压缩包解压后的文件

# 自定义为自己在阿里云的镜像和更改部分Chart配置，存放在文件中
[root@k8s-master01 ingress]# vim ingress-custom.yaml
controller:
  name: controller
  image:
    repository: registry.cn-hangzhou.aliyuncs.com/deemoprobe/ingress-nginx-controller
    tag: "1.1.1"
    digest: sha256:e16123f3932f44a2bba8bc3cf1c109cea4495ee271d6d16ab99228b58766d3ab

  dnsPolicy: ClusterFirstWithHostNet

  hostNetwork: true # 开启hostNetwork模式

  publishService:  # hostNetwork 模式下设置为false，通过节点IP地址上报ingress status数据
    enabled: false

  # 是否需要处理不带 ingressClass 注解或者 ingressClassName 属性的 Ingress 对象
  # 设置为 true 会在控制器启动参数中新增一个 --watch-ingress-without-class 标注
  watchIngressWithoutClass: false

  kind: DaemonSet

  tolerations:   # kubeadm 安装的集群默认情况下master是有污点，需要容忍这个污点才可以部署
  - key: "node-role.kubernetes.io/master"  # 生产环境不建议部署在master节点
    operator: "Equal"
    effect: "NoSchedule"

  nodeSelector:
    ingress: "true"  # 选择标签为ingress=true的节点进行部署

  service:  # HostNetwork 模式不需要创建service
    enabled: false

  admissionWebhooks: # 强烈建议开启 admission webhook
    enabled: true
    createSecretJob:
      resources:
        limits:
          cpu: 10m
          memory: 20Mi
        requests:
          cpu: 10m
          memory: 20Mi
    patchWebhookJob:
      resources:
        limits:
          cpu: 10m
          memory: 20Mi
        requests:
          cpu: 10m
          memory: 20Mi
    patch:
      enabled: true
      image:
        repository: registry.cn-hangzhou.aliyuncs.com/deemoprobe/kube-webhook-certgen
        tag: "1.1.1"
        digest: sha256:23a03c9c381fba54043d0f6148efeaf4c1ca2ed176e43455178b5c5ebf15ad70

defaultBackend:  # 启用并配置默认后端
  enabled: true
  name: defaultbackend
  image:
    repository: registry.cn-hangzhou.aliyuncs.com/deemoprobe/defaultbackend-amd64
    tag: "1.5"
    digest: sha256:5c51a4d6c2669c4fe765497153872ec6b0b12ce65f5cbadad6869c25d5197b3a
# 给想要部署ingress-nginx-controller的节点打上标签
[root@k8s-master01 ingress]# kubectl label node k8s-node01 ingress=true
[root@k8s-master01 ingress]# kubectl label node k8s-node02 ingress=true
[root@k8s-master01 ingress]# kubectl label node k8s-master01 ingress=true

# 查看Chart目录树构造，通常建议自定义文件去覆盖Chart values.yaml中的配置
# 可以根据目录树了解Chart构建时文件的分布和详细内容阅读
[root@k8s-master01 ingress]# tree ingress-nginx
ingress-nginx
├── CHANGELOG.md
├── Chart.yaml
├── ci
│   ├── controller-custom-ingressclass-flags.yaml
│   ├── daemonset-customconfig-values.yaml
│   ├── daemonset-customnodeport-values.yaml
│   ├── daemonset-extra-modules.yaml
│   ├── daemonset-headers-values.yaml
│   ├── daemonset-internal-lb-values.yaml
│   ├── daemonset-nodeport-values.yaml
│   ├── daemonset-podannotations-values.yaml
│   ├── daemonset-tcp-udp-configMapNamespace-values.yaml
│   ├── daemonset-tcp-udp-values.yaml
│   ├── daemonset-tcp-values.yaml
│   ├── deamonset-default-values.yaml
│   ├── deamonset-metrics-values.yaml
│   ├── deamonset-psp-values.yaml
│   ├── deamonset-webhook-and-psp-values.yaml
│   ├── deamonset-webhook-values.yaml
│   ├── deployment-autoscaling-behavior-values.yaml
│   ├── deployment-autoscaling-values.yaml
│   ├── deployment-customconfig-values.yaml
│   ├── deployment-customnodeport-values.yaml
│   ├── deployment-default-values.yaml
│   ├── deployment-extra-modules.yaml
│   ├── deployment-headers-values.yaml
│   ├── deployment-internal-lb-values.yaml
│   ├── deployment-metrics-values.yaml
│   ├── deployment-nodeport-values.yaml
│   ├── deployment-podannotations-values.yaml
│   ├── deployment-psp-values.yaml
│   ├── deployment-tcp-udp-configMapNamespace-values.yaml
│   ├── deployment-tcp-udp-values.yaml
│   ├── deployment-tcp-values.yaml
│   ├── deployment-webhook-and-psp-values.yaml
│   ├── deployment-webhook-resources-values.yaml
│   └── deployment-webhook-values.yaml
├── OWNERS
├── README.md
├── README.md.gotmpl
├── templates
│   ├── admission-webhooks
│   │   ├── job-patch
│   │   │   ├── clusterrolebinding.yaml
│   │   │   ├── clusterrole.yaml
│   │   │   ├── job-createSecret.yaml
│   │   │   ├── job-patchWebhook.yaml
│   │   │   ├── psp.yaml
│   │   │   ├── rolebinding.yaml
│   │   │   ├── role.yaml
│   │   │   └── serviceaccount.yaml
│   │   └── validating-webhook.yaml
│   ├── clusterrolebinding.yaml
│   ├── clusterrole.yaml
│   ├── controller-configmap-addheaders.yaml
│   ├── controller-configmap-proxyheaders.yaml
│   ├── controller-configmap-tcp.yaml
│   ├── controller-configmap-udp.yaml
│   ├── controller-configmap.yaml
│   ├── controller-daemonset.yaml
│   ├── controller-deployment.yaml
│   ├── controller-hpa.yaml
│   ├── controller-ingressclass.yaml
│   ├── controller-keda.yaml
│   ├── controller-poddisruptionbudget.yaml
│   ├── controller-prometheusrules.yaml
│   ├── controller-psp.yaml
│   ├── controller-rolebinding.yaml
│   ├── controller-role.yaml
│   ├── controller-serviceaccount.yaml
│   ├── controller-service-internal.yaml
│   ├── controller-service-metrics.yaml
│   ├── controller-servicemonitor.yaml
│   ├── controller-service-webhook.yaml
│   ├── controller-service.yaml
│   ├── default-backend-deployment.yaml
│   ├── default-backend-hpa.yaml
│   ├── default-backend-poddisruptionbudget.yaml
│   ├── default-backend-psp.yaml
│   ├── default-backend-rolebinding.yaml
│   ├── default-backend-role.yaml
│   ├── default-backend-serviceaccount.yaml
│   ├── default-backend-service.yaml
│   ├── dh-param-secret.yaml
│   ├── _helpers.tpl
│   ├── NOTES.txt
│   └── _params.tpl
└── values.yaml

4 directories, 84 files

[root@k8s-master01 ingress]# cd ingress-nginx/
# 使用ingress-custom.yaml覆盖Chart中values.yaml的对应字段值，创建ingress-nginx-controller
[root@k8s-master01 ingress-nginx]# helm install ingress-nginx . -n ingress-nginx -f ../ingress-custom.yaml
NAME: ingress-nginx
LAST DEPLOYED: Sat Jan 22 11:00:02 2022
NAMESPACE: ingress-nginx
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The ingress-nginx controller has been installed.
It may take a few minutes for the LoadBalancer IP to be available.
You can watch the status by running 'kubectl --namespace ingress-nginx get services -o wide -w ingress-nginx-controller'

An example Ingress that makes use of the controller:
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: example
    namespace: foo
  spec:
    ingressClassName: nginx
    rules:
      - host: www.example.com
        http:
          paths:
            - backend:
                service:
                  name: exampleService
                  port:
                    number: 80
              path: /
    # This section is only required if TLS is to be enabled for the Ingress
    tls:
      - hosts:
        - www.example.com
        secretName: example-tls

If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:

  apiVersion: v1
  kind: Secret
  metadata:
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
  type: kubernetes.io/tls
# 查看
[root@k8s-master01 ingress]# kubectl get po -n ingress-nginx -owide
NAME                                           READY   STATUS    RESTARTS   AGE    IP               NODE           NOMINATED NODE   READINESS GATES
ingress-nginx-controller-24khj                 1/1     Running   0          6m4s   192.168.43.187   k8s-node02     <none>           <none>
ingress-nginx-controller-86s8b                 1/1     Running   0          6m5s   192.168.43.183   k8s-master01   <none>           <none>
ingress-nginx-controller-jmkjt                 1/1     Running   0          6m4s   192.168.43.186   k8s-node01     <none>           <none>
ingress-nginx-defaultbackend-9db565b46-x4h8l   1/1     Running   0          6m4s   172.27.14.221    k8s-node02     <none>           <none>
[root@k8s-master01 ingress]# kubectl get svc -n ingress-nginx 
NAME                                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
ingress-nginx-controller-admission   ClusterIP   10.97.175.75   <none>        443/TCP   6m34s
ingress-nginx-defaultbackend         ClusterIP   10.99.84.57    <none>        80/TCP    6m34s
# 可以查看三个打了ingress=true标签的节点上的80/443端口均被ingress-nginx占用
# 访问他们的80端口，可以看到返回default backend 404
[root@k8s-master01 ingress]# curl k8s-node01
default backend - 404
[root@k8s-master01 ingress]# curl k8s-node02
default backend - 404
[root@k8s-master01 ingress]# curl k8s-master01
default backend - 404
# 创建Ingress实例
[root@k8s-master ingress]# vi ngdemo.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      app: my-nginx
  template:
    metadata:
      labels:
        app: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    app: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
    name: http
  selector:
    app: my-nginx
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-nginx
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: nginx.ingress.com  # 将域名映射到 my-nginx 服务
    http:
      paths:
      - path: /
        backend:
          serviceName: my-nginx  # 将所有请求发送到 my-nginx 服务的 80 端口
          servicePort: 80
[root@k8s-master ingress]# kubectl get svc
NAME                                                     TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)                      AGE
my-nginx                                                 ClusterIP      192.168.177.62   <none>          80/TCP                       12m
nginx-ingress-nginx-ingress-controller                   LoadBalancer   192.168.120.98   172.42.42.101   80:31051/TCP,443:30469/TCP   96m
nginx-ingress-nginx-ingress-controller-default-backend   ClusterIP      192.168.240.37   <none>          80/TCP                       96m
# 将域名配置到hosts
[root@k8s-master ingress]# vi /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.43.10   k8s-master
192.168.43.20   k8s-node1
172.42.42.101   nginx.ingress.com
[root@k8s-master ingress]# curl http://nginx.ingress.com
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</body>
</html>
```

> 说明：如果启用了service.enabled=true，查看services，发现external-ip处于Pending状态解决pending状态参考博客 <http://www.deemoprobe.com/yunv/ingress-pending/>或者将ingress-nginx目录下values.yaml中 type: LoadBalancer 字段改为 type: ClusterIP

下图显示了客户端是如果通过 Ingress Controller 连接到其中一个 Pod 的流程,客户端首先对 `nginx.ingress.com` 执行 DNS 解析,得到 Ingress Controller 所在节点的 IP,然后客户端向 Ingress Controller 发送 HTTP 请求,然后根据 Ingress 对象里面的描述匹配域名,找到对应的 Service 对象,并获取关联的 Endpoints 列表,将客户端的请求转发给其中一个 Pod.

![20210111153621](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20210111153621.png)

## Nginx-Ingress综合实例

通过部署两套Service来实现: HTTP代理访问/HTTPS代理访问/BasicAuth认证/Rewrite重写验证, 并为之分配不同的域名进行区分.

### 创建deploy-svc1

```shell
[root@k8s-master ingress]# vi deploy-svc1.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deploy1
  namespace: dev
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
        - name: http
          containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc1
  namespace: dev
spec:
  selector:
    app: myapp
    release: v1
  ports:
  - name: http
    port: 80
    targetPort: 80
[root@k8s-master ingress]# kubectl apply -f deploy-svc1.yaml 
deployment.apps/myapp-deploy1 created
service/myapp-svc1 created
[root@k8s-master ingress]# kubectl get deploy -o wide
NAME                                                     READY   UP-TO-DATE   AVAILABLE   AGE    CONTAINERS                                 IMAGES                                                           SELECTOR
myapp-deploy1                                            2/2     2            2           105s   myapp                                      registry.cn-beijing.aliyuncs.com/google_registry/myapp:v1        app=myapp,release=v1
...
[root@k8s-master ingress]# kubectl get rs -o wide
NAME                                                                DESIRED   CURRENT   READY   AGE     CONTAINERS                                 IMAGES                                                           SELECTOR
myapp-deploy1-6c468d6b6c                                            2         2         2       3m40s   myapp                                      registry.cn-beijing.aliyuncs.com/google_registry/myapp:v1        app=myapp,pod-template-hash=6c468d6b6c,release=v1
...
[root@k8s-master ingress]# kubectl get pod -o wide --show-labels
NAME                                                              READY   STATUS    RESTARTS   AGE     IP             NODE        NOMINATED NODE   READINESS GATES   LABELS
myapp-deploy1-6c468d6b6c-cq44z                                    1/1     Running   0          4m10s   172.16.36.86   k8s-node1   <none>           <none>            app=myapp,env=test,pod-template-hash=6c468d6b6c,release=v1
myapp-deploy1-6c468d6b6c-mzqwd                                    1/1     Running   0          4m10s   172.16.36.92   k8s-node1   <none>           <none>            app=myapp,env=test,pod-template-hash=6c468d6b6c,release=v1
# Curl访问
[root@k8s-master ingress]# curl 172.16.36.86
Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
[root@k8s-master ingress]# curl 172.16.36.92
Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
[root@k8s-master ingress]# curl 172.16.36.92/hostname.html
myapp-deploy1-6c468d6b6c-mzqwd
[root@k8s-master ingress]# curl 172.16.36.86/hostname.html
myapp-deploy1-6c468d6b6c-cq44z
# 查看SVC
[root@k8s-master ingress]# kubectl get svc 
NAME                                                     TYPE           CLUSTER-IP        EXTERNAL-IP     PORT(S)                      AGE
myapp-svc1                                               ClusterIP      192.168.226.120   <none>          80/TCP                       53s
...
# curl SVC, 可见Service后端两个Pod均可访问得到
[root@k8s-master ingress]# curl 192.168.226.120 
Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
[root@k8s-master ingress]# curl 192.168.226.120/hostname.html
myapp-deploy1-6c468d6b6c-mzqwd
[root@k8s-master ingress]# curl 192.168.226.120/hostname.html
myapp-deploy1-6c468d6b6c-cq44z
```

### 创建deploy-svc2

```shell
[root@k8s-master ingress]# vi deploy-svc2.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deploy2
  namespace: dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
      release: v2
  template:
    metadata:
      labels:
        app: myapp
        release: v2
        env: test
    spec:
      containers:
      - name: myapp
        image: registry.cn-beijing.aliyuncs.com/google_registry/myapp:v2
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc2
  namespace: dev
spec:
  selector:
    app: myapp
    release: v2
  ports:
  - name: http
    port: 80
    targetPort: 80
[root@k8s-master ingress]# kubectl apply -f deploy-svc2.yaml 
deployment.apps/myapp-deploy2 created
service/myapp-svc2 unchanged
[root@k8s-master ingress]# kubectl get deploy myapp-deploy2 -o wide
NAME            READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES                                                      SELECTOR
myapp-deploy2   2/2     2            2           47s   myapp        registry.cn-beijing.aliyuncs.com/google_registry/myapp:v2   app=myapp,release=v2
[root@k8s-master ingress]# kubectl get rs -o wide
NAME                                                                DESIRED   CURRENT   READY   AGE    CONTAINERS                                 IMAGES                                                           SELECTOR
myapp-deploy1-6c468d6b6c                                            2         2         2       13m    myapp                                      registry.cn-beijing.aliyuncs.com/google_registry/myapp:v1        app=myapp,pod-template-hash=6c468d6b6c,release=v1
myapp-deploy2-5fffdcccd5                                            2         2         2       71s    myapp                                      registry.cn-beijing.aliyuncs.com/google_registry/myapp:v2        app=myapp,pod-template-hash=5fffdcccd5,release=v2
...
[root@k8s-master ingress]# kubectl get po -o wide --show-labels -l "release=v2"
NAME                             READY   STATUS    RESTARTS   AGE     IP             NODE        NOMINATED NODE   READINESS GATES   LABELS
myapp-deploy2-5fffdcccd5-25qz4   1/1     Running   0          2m47s   172.16.36.87   k8s-node1   <none>           <none>            app=myapp,env=test,pod-template-hash=5fffdcccd5,release=v2
myapp-deploy2-5fffdcccd5-vpcj9   1/1     Running   0          2m47s   172.16.36.88   k8s-node1   <none>           <none>            app=myapp,env=test,pod-template-hash=5fffdcccd5,release=v2
[root@k8s-master ingress]# kubectl get svc -o wide | grep myapp-svc2
myapp-svc2                                               ClusterIP      192.168.114.169   <none>          80/TCP                       5m26s   app=myapp,release=v2
# Curl SVC
[root@k8s-master ingress]# curl 192.168.114.169
Hello MyApp | Version: v2 | <a href="hostname.html">Pod Name</a>
[root@k8s-master ingress]# curl 192.168.114.169/hostname.html
myapp-deploy2-5fffdcccd5-vpcj9
[root@k8s-master ingress]# curl 192.168.114.169/hostname.html
myapp-deploy2-5fffdcccd5-25qz4
```

### HTTP代理访问

```shell
[root@k8s-master ingress]# vi ingress-http.yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: nginx-http
  namespace: dev
spec:
  rules:
    - host: www.nginx-ingress.com
      http:
        paths:
        - path: /
          backend:
            serviceName: myapp-svc1
            servicePort: 80
    - host: info.nginx-ingress.com
      http:
        paths:
        - path: /
          backend:
            serviceName: myapp-svc2
            servicePort: 80
[root@k8s-master ingress]# kubectl apply -f ingress-http.yaml 
ingress.networking.k8s.io/nginx-http created
# 查看Ingress
[root@k8s-master ingress]# kubectl get ingress -o wide
NAME         CLASS    HOSTS                                          ADDRESS         PORTS   AGE
my-nginx     <none>   nginx.ingress.com                              192.168.43.20   80      85m
nginx-http   <none>   www.nginx-ingress.com,info.nginx-ingress.com   192.168.43.20   80      112s
# 查看nginx-ingress
[root@k8s-master ingress]# kubectl get po | grep ingress
nginx-ingress-nginx-ingress-controller-667cb64f9c-zqg2q           1/1     Running   0          175m
nginx-ingress-nginx-ingress-controller-default-backend-7ccrfv9m   1/1     Running   0          175m
# 进入nginx-ingress查看nginx.conf
[root@k8s-master ingress]# kubectl exec -it nginx-ingress-nginx-ingress-controller-667cb64f9c-zqg2q /bin/bash
I have no name!@nginx-ingress-nginx-ingress-controller-667cb64f9c-zqg2q:/$ cat /etc/nginx/nginx.conf
# 在这个配置文件里可以查看所有的nginx配置, 其中就包括www.nginx-ingress.com和info.nginx-ingress.com配置
# 查看ingress Pod
[root@k8s-master ingress]# kubectl get pod -o wide | grep ingress
nginx-ingress-nginx-ingress-controller-667cb64f9c-zqg2q           1/1     Running   0          3h16m   172.16.36.72   k8s-node1   <none>           <none>
nginx-ingress-nginx-ingress-controller-default-backend-7ccrfv9m   1/1     Running   0          3h16m   172.16.36.83   k8s-node1   <none>           <none>

# 将域名加入/etc/hosts
[root@k8s-master ingress]# vi /etc/hosts
[root@k8s-master ingress]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.43.10   k8s-master
192.168.43.20   k8s-node1
# 简单实例
172.42.42.101   nginx.ingress.com
# 综合实例
172.42.42.101   www.nginx-ingress.com info.nginx-ingress.com

# curl访问
[root@k8s-master ingress]# curl http://www.nginx-ingress.com
Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
[root@k8s-master ingress]# curl http://info.nginx-ingress.com
Hello MyApp | Version: v2 | <a href="hostname.html">Pod Name</a>
```

> 说明:实际环境中是对节点的外部IP进行访问, 只需要为节点分配好对应的外部IP即可, 然后在物理机上配置好hosts域名解析, 浏览器访问即可

#### 本地Windows系统下浏览器访问http

编辑文件 `C:\WINDOWS\System32\drivers\etc\hosts`

> 如果提示没有权限, 在文件属性里给User用户(当前所用用户)赋权即可编辑写入

hosts文件权限更改图:

![20210112100434](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20210112100434.png)

添加信息如下:

```shell
# kubernetes ingress
172.42.42.101   www.nginx-ingress.com info.nginx-ingress.com
```

浏览器访问结果图:

> 访问<http://www.nginx-ingress.com>和<http://www.nginx-ingress.com/hostname.html>

![20210112100558](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20210112100558.png)
![20210112101041](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20210112101041.png)

> 访问<http://info.nginx-ingress.com>和<http://info.nginx-ingress.com/hostname.html>

![20210112100628](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20210112100628.png)
![20210112101051](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20210112101051.png)

```shell
# 可以看到访问的hostname(即PodName)是当前在运行的PodName
[root@k8s-master cert]# kubectl get po
NAME                                                              READY   STATUS    RESTARTS   AGE
myapp-deploy1-6c468d6b6c-cq44z                                    1/1     Running   1          17h
myapp-deploy1-6c468d6b6c-mzqwd                                    1/1     Running   1          17h
myapp-deploy2-5fffdcccd5-25qz4                                    1/1     Running   1          17h
myapp-deploy2-5fffdcccd5-vpcj9                                    1/1     Running   1          17h
```

### HTTPS代理访问

#### 创建SSL证书

```shell
[root@k8s-master kubernetes]# mkdir cert;cd cert
[root@k8s-master cert]# openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/C=CN/ST=BJ/L=BeiJing/O=BTC/OU=MOST/CN=deemoprobe/emailAddress=ca@test.com"
Generating a 2048 bit RSA private key
........+++
........................................................................+++
writing new private key to 'tls.key'
-----
[root@k8s-master cert]# ls
tls.crt  tls.key
[root@k8s-master cert]# kubectl create secret tls tls-secret --key tls.key --cert tls.crt
secret/tls-secret created
```

#### 创建ingress https

```shell
[root@k8s-master cert]# cd ../ingress/
[root@k8s-master ingress]# vi ingress-https.yaml 
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: nginx-https
  namespace: dev
spec:
  tls:
    - hosts:
      - www.nginx-ingress.com
      - info.nginx-ingress.com
      secretName: tls-secret
  rules:
    - host: www.nginx-ingress.com
      http:
        paths:
        - path: /
          backend:
            serviceName: myapp-svc1
            servicePort: 80
    - host: info.nginx-ingress.com
      http:
        paths:
        - path: /
          backend:
            serviceName: myapp-svc2
            servicePort: 80
[root@k8s-master ingress]# kubectl apply -f ingress-https.yaml 
ingress.networking.k8s.io/nginx-https created
# 查看ingress
[root@k8s-master ingress]# kubectl get ingress -o wide
NAME          CLASS    HOSTS                                          ADDRESS         PORTS     AGE
my-nginx      <none>   nginx.ingress.com                              192.168.43.20   80        19h
nginx-http    <none>   www.nginx-ingress.com,info.nginx-ingress.com   192.168.43.20   80        17h
nginx-https   <none>   www.nginx-ingress.com,info.nginx-ingress.com   192.168.43.20   80, 443   53s
```

#### 本地Windows系统下浏览器访问https

> 编辑文件 C:\WINDOWS\System32\drivers\etc\hosts 添加对应的域名信息, 由于我用的和上面的http映射一样, 所以直接使用即可

访问时会提示不安全, 此时点击高级继续访问即可(由于该域名证书是自颁发而非CA机构颁发的证书,用来测试ingress,所以会提示不安全)

![20210112102800](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20210112102800.png)

浏览器访问结果图:

> 访问<https://www.nginx-ingress.com>和<https://www.nginx-ingress.com/hostname.html>

![20210112102830](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20210112102830.png)
![20210112102846](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20210112102846.png)

> 访问<https://info.nginx-ingress.com>和<https://info.nginx-ingress.com/hostname.html>

![20210112102915](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20210112102915.png)
![20210112102931](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20210112102931.png)

```shell
# 可以看到访问的hostname(即PodName)是当前在运行的PodName
[root@k8s-master cert]# kubectl get po
NAME                                                              READY   STATUS    RESTARTS   AGE
myapp-deploy1-6c468d6b6c-cq44z                                    1/1     Running   1          17h
myapp-deploy1-6c468d6b6c-mzqwd                                    1/1     Running   1          17h
myapp-deploy2-5fffdcccd5-25qz4                                    1/1     Running   1          17h
myapp-deploy2-5fffdcccd5-vpcj9                                    1/1     Running   1          17h
```

### Nginx-Ingress BasicAuth认证

#### 准备

```shell
[root@k8s-master ingress]# yum install -y httpd
[root@k8s-master ingress]# htpasswd -c auth deemoprobe
New password: #输入密码
Re-type new password: #确认密码
Adding password for user deemoprobe
# 会生成一个auth的文件
[root@k8s-master ingress]# ls
auth  deploy-svc1.yaml  deploy-svc2.yaml  ingress-https.yaml  ingress-http.yaml  mandatory.yaml  ngdemo.yaml
[root@k8s-master ingress]# cat auth 
deemoprobe:$apr1$7163wKOY$efGLFSlm6.dSHwIbaU6Ym0
# 创建secret
[root@k8s-master ingress]# kubectl create secret generic basic-auth --from-file=auth
secret/basic-auth created
[root@k8s-master ingress]# kubectl get secret basic-auth -o yaml
apiVersion: v1
data:
  auth: ZGVlbW9wcm9iZTokYXByMSQ3MTYzd0tPWSRlZkdMRlNsbTYuZFNId0liYVU2WW0wCg==
kind: Secret
metadata:
  creationTimestamp: "2021-01-12T02:35:33Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:auth: {}
      f:type: {}
    manager: kubectl-create
    operation: Update
    time: "2021-01-12T02:35:33Z"
  name: basic-auth
  namespace: dev
  resourceVersion: "793103"
  selfLink: /api/v1/namespaces/dev/secrets/basic-auth
  uid: c84f1541-9aba-4326-9142-3517bdcdfc34
type: Opaque
```

#### 创建ingress

```shell
[root@k8s-master ingress]# vi nginx-ingress-basicauth.yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: ingress-with-auth
  annotations:
    # type of authentication
    nginx.ingress.kubernetes.io/auth-type: basic
    # name of the secret that contains the user/password definitions
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    # message to display with an appropriate context why the authentication is required
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required - deemoprobe'
spec:
  rules:
  - host: auth.nginx-ingress.com
    http:
      paths:
      - path: /
        backend:
          serviceName: myapp-svc1
          servicePort: 80
[root@k8s-master ingress]# kubectl apply -f nginx-ingress-basicauth.yaml 
ingress.networking.k8s.io/ingress-with-auth created
[root@k8s-master ingress]# kubectl get ingress -o wide
NAME                CLASS    HOSTS                                          ADDRESS         PORTS     AGE
ingress-with-auth   <none>   auth.nginx-ingress.com                                         80        14s
my-nginx            <none>   nginx.ingress.com                              192.168.43.20   80        19h
nginx-http          <none>   www.nginx-ingress.com,info.nginx-ingress.com   192.168.43.20   80        18h
nginx-https         <none>   www.nginx-ingress.com,info.nginx-ingress.com   192.168.43.20   80, 443   19m
```

#### 浏览器访问auth

编辑文件 `C:\WINDOWS\System32\drivers\etc\hosts`

添加信息如下(在后面添加auth.nginx-ingress.com即可):

```shell
# kubernetes ingress
172.42.42.101   www.nginx-ingress.com info.nginx-ingress.com auth.nginx-ingress.com
```

> 访问<http://auth.nginx-ingress.com/>

输入账户密码进入myapp-svc1服务:
![20210112104413](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20210112104413.png)
![20210112104439](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20210112104439.png)

### Nginx-Ingress Rewrite重写验证

重写可以使用以下注解控制:

| 名称                                           | 描述                                                          | 值     |
| ---------------------------------------------- | ------------------------------------------------------------- | ------ |
| nginx.ingress.kubernetes.io/rewrite-target     | 必须重定向的目标URL                                           | String |
| nginx.ingress.kubernetes.io/ssl-redirect       | 指示位置部分是否只能由SSL访问(当Ingress包含证书时,默认为True) | Bool   |
| nginx.ingress.kubernetes.io/force-ssl-redirect | 即使Ingress没有启用TLS,也强制重定向到HTTPS                    | Bool   |
| nginx.ingress.kubernetes.io/app-root           | 定义应用程序根目录,Controller在“/”上下文中必须重定向该根目录  | String |
| nginx.ingress.kubernetes.io/use-regex          | 指示Ingress上定义的路径是否使用正则表达式                     | Bool   |

```shell
# 创建rewrite=ingress
[root@k8s-master ingress]# vi nginx-ingress-rewrite.yaml 
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: https://kubernetes.io/
  name: rewrite
  namespace: dev
spec:
  rules:
  - host: rewrite.nginx-ingress.com
    http:
      paths:
      - backend:
          serviceName: myapp-svc1
          servicePort: 80
[root@k8s-master ingress]# kubectl apply -f nginx-ingress-rewrite.yaml 
ingress.networking.k8s.io/rewrite created
[root@k8s-master ingress]# kubectl get ingress -o wide
NAME                CLASS    HOSTS                                          ADDRESS         PORTS     AGE
ingress-with-auth   <none>   auth.nginx-ingress.com                         192.168.43.20   80        14m
my-nginx            <none>   nginx.ingress.com                              192.168.43.20   80        19h
nginx-http          <none>   www.nginx-ingress.com,info.nginx-ingress.com   192.168.43.20   80        18h
nginx-https         <none>   www.nginx-ingress.com,info.nginx-ingress.com   192.168.43.20   80, 443   34m
rewrite             <none>   rewrite.nginx-ingress.com                                      80        12s
```

#### 浏览器访问rewrite

编辑文件 `C:\WINDOWS\System32\drivers\etc\hosts`

添加信息如下(在后面添加rewrite.nginx-ingress.com即可):

```shell
# kubernetes ingress
172.42.42.101   www.nginx-ingress.com info.nginx-ingress.com auth.nginx-ingress.com rewrite.nginx-ingress.com
```

> 访问<http://rewrite.nginx-ingress.com/>

访问后会跳转到Kubernetes官网:
![20210112105652](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20210112105652.png)
![20210112110219](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20210112110219.png)

#### CentOS本地系统访问

```shell
# 先添加本地hosts
[root@k8s-master ingress]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.43.10   k8s-master
192.168.43.20   k8s-node1
172.42.42.101  nginx.ingress.com
172.42.42.101   www.nginx-ingress.com info.nginx-ingress.com auth.nginx-ingress.com rewrite.nginx-ingress.com
# 可以看到已经进行了rewrite, 目标地址转向了https://kubernetes.io/
[root@k8s-master ingress]# curl -I http://rewrite.nginx-ingress.com 
HTTP/1.1 302 Moved Temporarily
Server: openresty/1.15.8.1
Date: Tue, 12 Jan 2021 02:59:51 GMT
Content-Type: text/html
Content-Length: 151
Connection: keep-alive
Location: https://kubernetes.io/
```
