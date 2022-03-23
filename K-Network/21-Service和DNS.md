# Kubernetes网络之Service和DNS

2021-0814

Service是一种Pod的服务发现策略，其他Pod可以通过这个Service访问到这个Service代理的Pod。

相对于Pod而言，它会有一个固定的名称，一旦创建就固定不变。包含服务的访问IP（Cluster IP）和端口`Kubernetes的Endpoints Controller会生成一个Endpoints对象, 记录 Endpoints=ClusterIP+Port`，通过`Selector`选择与之匹配的Pod。

## 定义

> 假设存在一个或一组打有"app=myapp"标签的pod，Pod对外暴露的端口是9376，那么可以创建Service去代理这些Pod

```bash
# 单个端口代理示例
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: myapp         # 选择带有该标签的Pod
  ports:
    - protocol: TCP    # UDP TCP SCTP default: TCP
      port: 8080       # Service自己的端口
      targetPort: 9376 # 后端应用Pod暴露的端口
      # name: http     # Service端口的名称
# 多个端口代理示例
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: myapp
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 9376
    - name: https
      protocol: TCP
      port: 443
      targetPort: 9377
```

> 说明：Service 能够将一个接收 port 映射到任意的 targetPort。 默认情况下，targetPort 将被设置为与 port 字段相同的值。

## 代理模式

在 Kubernetes 集群中，每个 Node 运行一个 `kube-proxy`进程。`kube-proxy`负责为Service实现了一种VIP形式的代理，而不是`ExternalName`的形式。

### iptables代理模式

在iptables模式中，kube-proxy 会监视 Kubernetes 控制节点对 Service 对象和 Endpoints 对象的添加和移除。 对每个 Service，它会配置`iptables`规则，从而捕获到达该 Service 的`clusterIP`和端口的请求，进而将请求重定向到 Service 的一组后端中的某个 Pod 上面。 对于每个 Endpoints 对象，它也会配置`iptables`规则，这个规则会选择一个后端组合。

默认的策略是，kube-proxy 在 iptables 模式下随机选择一个后端。

使用 iptables 处理流量具有较低的系统开销，因为流量由`Linux netfilter`处理，而无需在用户空间和内核空间之间切换。

![20220119161352](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220119161352.png)

### IPVS代理模式

在ipvs模式中，kube-proxy 监视 Kubernetes的Service和Endpoint，调用`netlink`接口相应地创建 IPVS 规则，并定期将 IPVS 规则与Kubernetes的Service和Endpoint同步。该控制循环可确保IPVS状态与所需状态匹配。访问服务时，IPVS将流量定向到后端Pod之一。

IPVS代理模式基于类似于iptables模式的`netfilter`，但是使用哈希表作为基础数据结构，并且在内核空间中工作。这意味着，与 iptables 模式下的 kube-proxy 相比，IPVS 模式下的 kube-proxy 重定向通信的延迟要短，并且在同步代理规则时具有更好的性能。与其他代理模式相比，IPVS 模式还支持更高的网络流量吞吐量。

IPVS 提供了更多选项来平衡后端 Pod 的流量：

- `rr`：轮替（Round-Robin）
- `lc`：最少链接（Least Connection），即打开链接数量最少者优先
- `dh`：目标地址哈希（Destination Hashing）
- `sh`：源地址哈希（Source Hashing）
- `sed`：最短预期延迟（Shortest Expected Delay）
- `nq`：从不排队（Never Queue）

> 说明：要在 IPVS 模式下运行 kube-proxy，必须在启动 kube-proxy 之前使 IPVS 在节点上可用（启用IPVS可以参考[高可用集群安装文档](http://www.deemoprobe.com/yunv/kuberneteskubadm/#ALL-2)）。当 kube-proxy 以 IPVS 代理模式启动时，它将验证 IPVS 内核模块是否可用。 如果未检测到 IPVS 内核模块，则 kube-proxy 将退回到以 iptables 代理模式运行。

![20220119161939](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220119161939.png)

这些代理模式绑定Service IP的流量：将Port代理到适当的后端服务。

如果要确保每次都将来自特定客户端的连接传递到同一Pod（即会话保持），则可以通过设置`service.spec.sessionAffinity: ClientIP`（默认值是 `None`），来基于客户端的 IP 地址选择会话关联。还可以通过设置`service.spec.sessionAffinityConfig.clientIP.timeoutSeconds`来配置最大会话停留时间。（默认值为 10800 秒，即 3 小时）

## 流量转发策略

### 外部流量

可以通过设置`spec.externalTrafficPolicy`字段来控制来自于外部的流量是如何路由的。可选值有`Cluster`和`Local`。字段设为`Cluster`会将外部流量路由到所有就绪的Endpoint，设为`Local`只会路由到当前节点上就绪的Endpoint。如果流量策略设置为`Local`，而当前节点上没有就绪的Endpoint，kube-proxy不会转发请求相关服务的任何流量。

### 内部流量

可以通过设置`spec.internalTrafficPolicy`字段来控制内部来源的流量是如何转发的。可设置的值有`Cluster`和`Local`。将字段设置为`Cluster`会将内部流量路由到所有就绪Endpoint，设置为`Local`只会路由到当前节点上就绪的Endpoint。如果流量策略是`Local`，而当前节点上没有就绪的Endpoint，那么 kube-proxy 会丢弃流量。

## 服务发现

Kubernetes 支持两种基本的服务发现模式 —— 环境变量和 DNS。

- 环境变量: Pod创建的时候，服务的ip和port等信息会以环境变量的形式注入到pod里
- DNS: Service创建成功后，会在dns服务器里导入一些记录，想要访问某个服务，通过dns服务器解析出对应的ip和port，从而实现服务访问

### 环境变量

当 Pod 运行在 Node 上，kubelet 会为每个活跃的 Service 添加一组环境变量。 支持 {SVCNAME}_SERVICE_HOST 和 {SVCNAME}_SERVICE_PORT 等变量。这里 Service 的名称需大写，横线被转换成下划线。

例如有一个名称为 nginx-service 的 Service 暴露了 TCP 端口8180，同时给它分配了`Cluster IP`地址 10.96.0.11，这个 Service 生成了如下环境变量：

```bash
NGINX_SERVICE_SERVICE_HOST=10.96.0.11
NGINX_SERVICE_PORT_8180_TCP_ADDR=10.96.0.11
KUBERNETES_PORT_443_TCP_PROTO=tcp
NGINX_SERVICE_PORT_8180_TCP_PORT=8180
NGINX_SERVICE_PORT_8180_TCP_PROTO=tcp
NGINX_SERVICE_SERVICE_PORT=8180
NGINX_SERVICE_PORT=tcp://10.96.0.11:8180
NGINX_SERVICE_PORT_8180_TCP=tcp://10.96.0.11:8180
```

```bash
# 实例
[root@k8s-master01 ~]# mkdir network
[root@k8s-master01 ~]# cd network/
[root@k8s-master01 network]# vim nginx-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  labels:
    app: nginx
spec:
  replicas: 2
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
# 配置Service
[root@k8s-master01 network]# vim nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  labels:
    name: nginx-service
spec:
  ports:
  - port: 8180
    targetPort: 80
  selector:
    app: nginx
[root@k8s-master01 network]# kubectl apply -f nginx-deploy.yaml
deployment.apps/nginx-deploy created
[root@k8s-master01 network]# kubectl apply -f nginx-service.yaml
service/nginx-service created
[root@k8s-master01 network]# kubectl get deployments.apps,svc
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deploy   2/2     2            2           40s

NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/nginx-service   ClusterIP   10.105.254.62   <none>        8180/TCP   33s
[root@k8s-master01 network]# kubectl get po
NAME                           READY   STATUS      RESTARTS      AGE
busybox                        1/1     Running     3 (38m ago)   4h46m
look-svc-env                   0/1     Completed   0             18s
[root@k8s-master01 network]# vim look-svc-env.yaml
apiVersion: v1
kind: Pod
metadata:
  name: look-svc-env
spec:
  containers:
  - name: look-svc-env
    image: busybox
    command: ["/bin/sh", "-c", "env"]
[root@k8s-master01 network]# kubectl apply -f look-svc-env.yaml 
pod/look-svc-env created
# 可以查看到nginx-service环境变量已经写入
[root@k8s-master01 network]# kubectl logs look-svc-env 
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_SERVICE_PORT=443
HOSTNAME=look-svc-env
SHLVL=1
HOME=/root
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
NGINX_SERVICE_SERVICE_HOST=10.105.254.62
KUBERNETES_PORT_443_TCP_PORT=443
NGINX_SERVICE_PORT_8180_TCP_ADDR=10.105.254.62
KUBERNETES_PORT_443_TCP_PROTO=tcp
NGINX_SERVICE_PORT_8180_TCP_PORT=8180
NGINX_SERVICE_PORT_8180_TCP_PROTO=tcp
NGINX_SERVICE_PORT=tcp://10.105.254.62:8180
NGINX_SERVICE_SERVICE_PORT=8180
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_SERVICE_HOST=10.96.0.1
PWD=/
NGINX_SERVICE_PORT_8180_TCP=tcp://10.105.254.62:8180
# nginx-service停掉后，nginx-service相关的环境变量消失
```

### DNS

支持集群的 DNS 服务器（例如 CoreDNS）监视 Kubernetes API 中的Service，并为每个Service创建一组 DNS 记录。如果在整个集群中都启用了DNS，则所有 Pod 都应该能够通过其 DNS 名称自动解析服务。

例如，如果你在 Kubernetes 命名空间`my-ns`中有一个名为`my-service`的Service，则控制节点和DNS服务共同为`my-service.my-ns`创建DNS记录。`my-ns`命名空间中的Pod可以直接用名称`my-service`来找到服务（当然使用`my-service.my-ns`也可以）。

Kubernetes 从 v1.11 开始可以使用 CoreDNS 来提供命名服务，并从 v1.13 开始成为默认 DNS 服务。CoreDNS 的特点是效率更高，资源占用率更小，推荐使用 CoreDNS 替代 kube-dns 为集群提供 DNS 服务。CoreDNS基本架构如下:

![20201208151150](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20201208151150.png)

如果需要定制DNS服务,可参考下面官方给的方案:<https://kubernetes.io/zh/docs/tasks/administer-cluster/dns-custom-nameservers/>

## Headless Service

有时不需要或不想要负载均衡，以及单独的 Service IP。可以通过指定spec下面的`clusterIP: None`来创建 Headless Service。

无头服务并不会分配 Cluster IP，kube-proxy 不会处理它们，而且平台也不会为它们进行负载均衡和路由。DNS如何实现自动配置，依赖于 Service 是否定义了`selector`。

对于定义了`selector`的无头服务，Endpoint 控制器在 API 中创建了 Endpoints 记录，并且修改 DNS 配置返回 A 记录（IP 地址），通过这个地址直接到达 Service 的后端 Pod 上。

对于没有定义`selector`的无头服务，Endpoint 控制器不会创建 Endpoints 记录。DNS系统会查找和配置：

- 对于 ExternalName 类型的服务，查找其 CNAME 记录
- 对所有其他类型的服务，查找与 Service 名称相同的任何 Endpoints 的记录

## Service发布类型

Kubernetes 允许指定你所需要的 Service 类型，默认是 ClusterIP，可以自定义`spec`字段下的`type`字段。`type`的取值以及行为如下：

- `ClusterIP`：通过集群的内部 IP 暴露服务，选择该值时服务只能够在集群内部访问。
- `NodePort`：通过每个节点上的 IP 和静态端口（NodePort）暴露服务。NodePort服务会路由到自动创建的ClusterIP服务。通过请求`<节点 IP>:<节点端口>`，NodePort端口范围默认是30000-32767。可以从集群的外部访问。
- `LoadBalancer`：使用云提供商的负载均衡器向外部暴露服务。外部负载均衡器可以将流量路由到自动创建的NodePort服务和ClusterIP服务上。
- `ExternalName`：通过返回CNAME（别名）和对应值，可以将服务映射到 externalName 字段的内容（例如，foo.bar.example.com）。无需创建任何类型代理。

> 说明：kube-dns 1.7 及以上版本或者 CoreDNS 0.0.8 及以上版本才能使用 ExternalName 类型。

```bash
# ClusterIP类型，type可以不指定
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: myapp
  ports:
      # 默认情况下，为了方便起见，`targetPort` 被设置为与 `port` 字段相同的值。
    - port: 80
      targetPort: 80

# NodePort类型
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
      # 默认情况下，为了方便起见，`targetPort` 被设置为与 `port` 字段相同的值。
    - port: 80
      targetPort: 80
      # 可选字段，不指定的话会自动设置为30000-32767中的一个
      nodePort: 30007

# LoadBalancer类型，status.loadBalancer字段指定云厂商提供的负载均衡地址
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  clusterIP: 10.96.0.239
  type: LoadBalancer
status:
  loadBalancer:
    ingress:
      - ip: 192.0.2.127

# ExternalName类型
# 访问nginx-externalname.test.svc.cluster.local被重定向到www.baidu.com
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx-externalname
  name: nginx-externalname
  namespace: test
spec:
  type: ExternalName
  externalName: www.baidu.com
```

## 外部IP

如果外部IP路由到集群中一个或多个Node上，Kubernetes Service 会被暴露给这些`externalIP`。通过外部 IP（作为目的 IP 地址）进入到集群，流量将会被路由到 Service 的 Endpoint 上。根据 Service 的规定，externalIPs 可以同任意的 ServiceType 来一起指定。 在下面的例子中，my-service 可以通过 "80.11.12.10:80"(externalIP:port)被客户端访问。

```bash
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: myapp
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 9376
  externalIPs:
    - 80.11.12.10
```

## Pod DNS

DNS策略可以逐个Pod来设定。目前 Kubernetes 支持以下特定 Pod 的 DNS 策略。这些策略可以在Pod的`spec.dnsPolicy`字段设置：

- "Default": Pod 从运行所在的节点继承名称解析配置
- "ClusterFirst": 与配置的集群域后缀不匹配的任何 DNS 查询（例如 "www.kubernetes.io"） 都将转发到从节点继承的上游名称服务器。集群管理员可能配置了额外的存根域和上游 DNS 服务器
- "ClusterFirstWithHostNet"：对于以 hostNetwork 方式运行的 Pod，应显式设置其 DNS 策略 "ClusterFirstWithHostNet"
- "None": 此设置允许 Pod 忽略 Kubernetes 环境中的 DNS 设置。Pod 会使用其`dnsConfig`字段所提供的 DNS 设置

> 说明： "Default" 不是默认的 DNS 策略。如果未明确指定 dnsPolicy，则使用 "ClusterFirst"。

Pod 的 DNS 配置可让用户对 Pod 的 DNS 设置进行更多控制。

dnsConfig 字段是可选的，它可以与任何 dnsPolicy 设置一起使用。 但是，当 Pod 的 dnsPolicy 设置为 "None" 时，必须指定 dnsConfig 字段。

用户可以在 dnsConfig 字段中指定以下属性：

- nameservers：将用作于 Pod 的 DNS 服务器的 IP 地址列表。 最多可以指定 3 个 IP 地址。当 Pod 的 dnsPolicy 设置为 "None" 时， 列表必须至少包含一个 IP 地址，否则此属性是可选的
- searches：用于在 Pod 中查找主机名的 DNS 搜索域的列表。此属性是可选的, Kubernetes 最多允许 6 个搜索域
- options：可选的对象列表，其中每个对象可能具有 name 属性（必需）和 value 属性（可选）

```bash
# 创建自定义DNS的Pod
[root@k8s-master01 network]# vim pod_dns.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.18.0
    ports:
    - containerPort: 80
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
      - 8.8.8.8
    searches:
      - default.svc.cluster-domain.example
      - cluster-domain.example
    options:
      - name: pod_num
        value: "1"
[root@k8s-master01 network]# kubectl apply -f pod_dns.yaml 
pod/nginx created
# 查看DNS信息
[root@k8s-master01 network]# kubectl exec nginx -- cat /etc/resolv.conf
nameserver 8.8.8.8
search default.svc.cluster-domain.example cluster-domain.example
options pod_num:1

# Pod访问验证
# 创建Service来暴露Pod
[root@k8s-master01 network]# cat nginx-service.yaml 
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  labels:
    name: nginx-service
spec:
  ports:
  - port: 8180
    targetPort: 80
  selector:
    app: nginx
[root@k8s-master01 network]# kubectl apply -f nginx-service.yaml 
service/nginx-service created
[root@k8s-master01 network]# kubectl get svc
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
nginx-service   ClusterIP   10.97.188.44   <none>        8180/TCP   75s
[root@k8s-master01 network]# kubectl exec busybox -- nslookup 10.97.188.44
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      10.97.188.44
Address 1: 10.97.188.44 nginx-service.default.svc.cluster.local
[root@k8s-master01 network]# kubectl exec busybox -- wget -q -O- nginx-service.default.svc.cluster.local:8180
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
