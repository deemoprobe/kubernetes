# Kubernetes网络之Ingress

> 本文通过Helm部署Nginx-Ingress-Controller.

Ingress 是对集群中服务的外部访问进行管理的 API 对象,典型的访问方式是 HTTP和HTTPS.

Ingress 可以提供负载均衡、SSL 和基于名称的虚拟托管.

必须具有 ingress 控制器[例如 ingress-nginx]才能满足 Ingress 的要求.仅创建 Ingress 资源无效.

## 1. Ingress原理

Ingress 公开了从集群外部到集群内 services 的 HTTP 和 HTTPS 路由. 流量路由由 Ingress 资源上定义的规则控制.

![20210112135036](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20210112135036.png)

可以将 Ingress 配置为提供服务外部可访问的 URL、负载均衡流量、 SSL / TLS,以及提供基于名称的虚拟主机.Ingress 控制器 通常负责通过负载均衡器来实现 Ingress,尽管它也可以配置边缘路由器或其他前端来帮助处理流量.

Ingress 不会公开任意端口或协议.若将 HTTP 和 HTTPS 以外的服务公开到 Internet 时,通常使用 Service.Type=NodePort 或者 Service.Type=LoadBalancer 类型的服务.

Nginx Ingress架构示意图:

![Ingress-Nginx](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/Ingress-Nginx.jpg)

> 必须具有 Ingress 控制器 才能满足 Ingress 的要求。 仅创建 Ingress 资源本身没有任何效果。

## 2. 部署Helm 3.4

helm通过打包的方式,支持发布的版本管理和控制,很大程度上简化了Kubernetes应用的部署和管理.

Helm本质就是让k8s的应用管理（Deployment、Service等）可配置,能动态生成.通过动态生成K8S资源清单文件（deployment.yaml、service.yaml）.然后kubectl自动调用K8S资源部署.

> 说明: Helm3.x 版本已经不需要再安装tiller(之前老版本中的Helm仓库的服务端), 直接安装配置好仓库就可以使用了

```shell
# 在官方(https://github.com/helm/helm/releases)下载想要的的版本, 当前(2021-01-11)最新稳定版 V3.4.2
# 解压并配置
[root@k8s-master ~]# tar -zxvf helm-v3.4.2-linux-amd64.tar.gz
[root@k8s-master ~]# mv linux-amd64/helm /usr/local/bin/helm
[root@k8s-master ~]# helm version
version.BuildInfo{Version:"v3.4.2", GitCommit:"23dd3af5e19a02d4f4baa5b2f242645a1a3af629", GitTreeState:"clean", GoVersion:"go1.14.13"}
# 添加阿里仓库
[root@k8s-master ~]# helm repo add apphub https://apphub.aliyuncs.com
"apphub" has been added to your repositories
[root@k8s-master ~]# helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "apphub" chart repository
Update Complete. ⎈Happy Helming!⎈
[root@k8s-master ~]# helm repo list
NAME    URL                        
apphub  https://apphub.aliyuncs.com
```

## 3. Helm部署 Nginx-Ingress-Controller

```shell
# 切换到dev这个namespace下
[root@k8s-master ~]# kubens dev
Context "kubernetes-admin@kubernetes" modified.
Active namespace is "dev".
[root@k8s-master ~]# kubens -c
dev
[root@k8s-master ~]# kubectl get po
No resources found in dev namespace.
# 查找Ingress资源包
[root@k8s-master ~]# helm search repo apphub | grep ingress
apphub/aws-alb-ingress-controller       0.1.13          v1.1.5                          A Helm chart for AWS ALB Ingress Controller       
apphub/gce-ingress                      1.2.0           1.4.0                           A GCE Ingress Controller                          
apphub/haproxy-ingress                  0.0.22          0.7.2                           Ingress controller implementation for haproxy l...
apphub/ingressmonitorcontroller         1.0.48          1.0.47                          IngressMonitorController chart that runs on kub...
apphub/nginx-ingress                    1.30.3          0.28.0                          An nginx Ingress controller that uses ConfigMap...
apphub/nginx-ingress-controller         5.3.4           0.29.0                          Chart for the nginx Ingress controller            
[root@k8s-master kubernetes]# helm install nginx-ingress apphub/nginx-ingress-controller
NAME: nginx-ingress
LAST DEPLOYED: Mon Jan 11 13:50:42 2021
NAMESPACE: dev
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
** Please be patient while the chart is being deployed **

The nginx-ingress controller has been installed.
It may take a few minutes for the LoadBalancer IP to be available.
You can watch the status by running 'kubectl --namespace dev get services -o wide -w nginx-ingress-nginx-ingress-controller'

An example Ingress that makes use of the controller:

  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    annotations:
      kubernetes.io/ingress.class: nginx
    name: example
    namespace: foo
  spec:
    rules:
      - host: www.example.com
        http:
          paths:
            - backend:
                serviceName: exampleService
                port: 80
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
# 安装提示查看services
[root@k8s-master manifests]# kubectl --namespace dev get services -o wide -w nginx-ingress-nginx-ingress-controller
NAME                                     TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)                      AGE   SELECTOR
nginx-ingress-nginx-ingress-controller   LoadBalancer   192.168.120.98   172.42.42.101   80:31051/TCP,443:30469/TCP   80m   app=nginx-ingress-controller,component=controller,release=nginx-ingress
# 以YAML形式查看配置
[root@k8s-master manifests]# kubectl get svc nginx-ingress-nginx-ingress-controller -o yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    meta.helm.sh/release-name: nginx-ingress
    meta.helm.sh/release-namespace: dev
  creationTimestamp: "2021-01-11T05:50:43Z"
  labels:
    app: nginx-ingress-controller
    app.kubernetes.io/managed-by: Helm
    chart: nginx-ingress-controller-5.3.4
    component: controller
    heritage: Helm
    release: nginx-ingress
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:meta.helm.sh/release-name: {}
          f:meta.helm.sh/release-namespace: {}
        f:labels:
          .: {}
          f:app: {}
          f:app.kubernetes.io/managed-by: {}
          f:chart: {}
          f:component: {}
          f:heritage: {}
          f:release: {}
      f:spec:
        f:externalTrafficPolicy: {}
        f:ports:
          .: {}
          k:{"port":80,"protocol":"TCP"}:
            .: {}
            f:name: {}
            f:port: {}
            f:protocol: {}
            f:targetPort: {}
          k:{"port":443,"protocol":"TCP"}:
            .: {}
            f:name: {}
            f:port: {}
            f:protocol: {}
            f:targetPort: {}
        f:selector:
          .: {}
          f:app: {}
          f:component: {}
          f:release: {}
        f:sessionAffinity: {}
        f:type: {}
    manager: Go-http-client
    operation: Update
    time: "2021-01-11T05:50:43Z"
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:status:
        f:loadBalancer:
          f:ingress: {}
    manager: controller
    operation: Update
    time: "2021-01-11T06:19:47Z"
  name: nginx-ingress-nginx-ingress-controller
  namespace: dev
  resourceVersion: "755241"
  selfLink: /api/v1/namespaces/dev/services/nginx-ingress-nginx-ingress-controller
  uid: 279f27eb-0b77-4693-a0c5-4e77633acd86
spec:
  clusterIP: 192.168.120.98
  externalTrafficPolicy: Cluster
  ports:
  - name: http
    nodePort: 31051
    port: 80
    protocol: TCP
    targetPort: http
  - name: https
    nodePort: 30469
    port: 443
    protocol: TCP
    targetPort: https
  selector:
    app: nginx-ingress-controller
    component: controller
    release: nginx-ingress
  sessionAffinity: None
  type: LoadBalancer
status:
  loadBalancer:
    ingress:
    - ip: 172.42.42.101
[root@k8s-master manifests]# kubectl get po -o wide
NAME                                                              READY   STATUS    RESTARTS   AGE   IP             NODE        NOMINATED NODE   READINESS GATES
nginx-ingress-nginx-ingress-controller-667cb64f9c-zqg2q           1/1     Running   0          72m   172.16.36.72   k8s-node1   <none>           <none>
nginx-ingress-nginx-ingress-controller-default-backend-7ccrfv9m   1/1     Running   0          72m   172.16.36.83   k8s-node1   <none>           <none>
[root@k8s-master manifests]# kubectl get svc
NAME                                                     TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)                      AGE
nginx-ingress-nginx-ingress-controller                   LoadBalancer   192.168.120.98   172.42.42.101   80:31051/TCP,443:30469/TCP   74m
nginx-ingress-nginx-ingress-controller-default-backend   ClusterIP      192.168.240.37   <none>          80/TCP                       74m
# LoadBalancer   192.168.120.98   172.42.42.101 
# ClusterIP      192.168.240.37
# PodIP          172.16.36.72
# NodeIP+Port    192.168.43.20:31051
# 上面这些均可访问Nginx服务
[root@k8s-master manifests]# curl 172.42.42.101
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</body>
</html>
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

下图显示了客户端是如果通过 Ingress Controller 连接到其中一个 Pod 的流程,客户端首先对 `nginx.ingress.com` 执行 DNS 解析,得到 Ingress Controller 所在节点的 IP,然后客户端向 Ingress Controller 发送 HTTP 请求,然后根据 Ingress 对象里面的描述匹配域名,找到对应的 Service 对象,并获取关联的 Endpoints 列表,将客户端的请求转发给其中一个 Pod.

![20210111153621](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20210111153621.png)

## 4. Nginx-Ingress综合实例

通过部署两套Service来实现: HTTP代理访问/HTTPS代理访问/BasicAuth认证/Rewrite重写验证, 并为之分配不同的域名进行区分.

### 4.1. 创建deploy-svc1

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

### 4.2. 创建deploy-svc2

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

### 4.3. HTTP代理访问

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

#### 4.3.1. 本地Windows系统下浏览器访问http

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

### 4.4. HTTPS代理访问

#### 4.4.1. 创建SSL证书

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

#### 4.4.2. 创建ingress https

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

#### 4.4.3. 本地Windows系统下浏览器访问https

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

### 4.5. Nginx-Ingress BasicAuth认证

#### 4.5.1. 准备

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

#### 4.5.2. 创建ingress

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

#### 4.5.3. 浏览器访问auth

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

### 4.6. Nginx-Ingress Rewrite重写验证

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

#### 4.6.1. 浏览器访问rewrite

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

#### 4.6.2. CentOS本地系统访问

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
