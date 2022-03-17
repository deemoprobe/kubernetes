# Kubernetes安全机制之RBAC

2021-0812

Kubernetes通过一系列安全机制来确保集群的安全，包括API Server的认证授权（Authentication/Authorization）、准入控制机制（Admission Control）、敏感信息的保护机制（Secret/Service_Account）等。

## 认证Authentication

- **HTTP Token认证：**通过一个Token来识别合法用户。HTTP Token的认证是用一个很长的特殊编码方式的并且难以被模仿的字符串来表达客户的一种方式。每一个Token对应一个用户名，存储在API Server能访问的文件中，当客户端发起API调用请求时，需要在HTTP Header里植入Token
- **HTTP Base认证：**通过`用户名+密码`的方式认证。用base64算法进行编码后的字符串放在HTTP Request中的Heather Authorization 域里发送给服务端，服务端收到后进行解码，获取用户名和密码
- **最严格的HTTPS证书认证：**基于CA根证书签名的客户端身份认证方式

## 授权Authorization

认证只是确认通信的双方都是可信的，认证后允许相互通信。而授权是确定请求方有哪些资源的权限。API Server目前支持如下几种授权策略（通过API Server的启动参数`--authorization-mode`设置）

- AlwaysDeny：表示拒绝所有请求，仅用于测试
- AlwaysAllow：表示允许所有请求，一般不建议
- Node：节点授权是一种特殊用途的授权模式，专门授权由 kubelet 发出的 API 请求
- Webhook：是一种 HTTP 回调模式，允许使用远程 REST 端点管理授权
- ABAC（Attribute-Based Access Control）：基于属性的访问控制，表示使用用户配置的授权规则对用户请求进行匹配和控制
- RBAC（Role-Based Access Control）：基于角色的访问控制，默认使用该规则

## RBAC授权模式

RBAC基于角色的访问控制，在Kubernetes 1.5中引入，现为默认授权标准。相对其他访问控制方式，拥有如下优势：

- 对集群中的资源和非资源均拥有完整的覆盖
- RBAC由几个API对象完成，同其他API对象一样，可以用kubectl或API进行操作
- 可以在运行时进行操作，无需重启API Server

```bash
# 二进制集群，kube-apiserver以守护进程方式运行，可以查看进程详情或service配置文件
# 查看授权模式为--authorization-mode=Node,RBAC
[root@k8s-master01 ~]# systemctl status kube-apiserver.service -l
● kube-apiserver.service - Kubernetes API Server
   Loaded: loaded (/usr/lib/systemd/system/kube-apiserver.service; enabled; vendor preset: disabled)
   Active: active (running) since Thu 2022-03-17 10:03:31 CST; 1min 21s ago
     Docs: https://github.com/kubernetes/kubernetes
 Main PID: 1001 (kube-apiserver)
    Tasks: 14
   Memory: 397.7M
   CGroup: /system.slice/kube-apiserver.service
           └─1001 /usr/local/bin/kube-apiserver ... --authorization-mode=Node,RBAC ....
# 如果是kubeadm安装的集群，kube-apiserver等组件以pod方式运行在kube-system这个namespace中
```

RBAC所声明的四种资源对象（Role、ClusterRole、RoleBinding 和 ClusterRoleBinding）用户可以像与其他 API 资源交互一样，通过 kubectl API 调用与这些资源交互。

![20210107104749](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20210107104749.png)

```bash
# 查看四个资源对象的apiVersion和描述信息，已做精简
[root@k8s-master01 ~]# kubectl explain role
KIND:     Role
VERSION:  rbac.authorization.k8s.io/v1

DESCRIPTION:
     Role is a namespaced, logical grouping of PolicyRules that can be
     referenced as a unit by a RoleBinding.

FIELDS:
   apiVersion   <string>
   kind <string>
   metadata     <Object>
   rules        <[]Object>
     Rules holds all the PolicyRules for this Role

[root@k8s-master01 ~]# kubectl explain rolebinding
KIND:     RoleBinding
VERSION:  rbac.authorization.k8s.io/v1

DESCRIPTION:
     RoleBinding references a role, but does not contain it. It can reference a
     Role in the same namespace or a ClusterRole in the global namespace. It
     adds who information via Subjects and namespace information by which
     namespace it exists in. RoleBindings in a given namespace only have effect
     in that namespace.

FIELDS:
   apiVersion   <string>
   kind <string>
   metadata     <Object>
   roleRef      <Object> -required-
     RoleRef can reference a Role in the current namespace or a ClusterRole in
     the global namespace. If the RoleRef cannot be resolved, the Authorizer
     must return an error.
   subjects     <[]Object>
     Subjects holds references to the objects the role applies to.

[root@k8s-master01 ~]# kubectl explain clusterrole
KIND:     ClusterRole
VERSION:  rbac.authorization.k8s.io/v1

DESCRIPTION:
     ClusterRole is a cluster level, logical grouping of PolicyRules that can be
     referenced as a unit by a RoleBinding or ClusterRoleBinding.

FIELDS:
   aggregationRule      <Object>
     AggregationRule is an optional field that describes how to build the Rules
     for this ClusterRole. If AggregationRule is set, then the Rules are
     controller managed and direct changes to Rules will be stomped by the
     controller.
   apiVersion   <string>
   kind <string>
   metadata     <Object>
   rules        <[]Object>
     Rules holds all the PolicyRules for this ClusterRole

[root@k8s-master01 ~]# kubectl explain clusterrolebinding
KIND:     ClusterRoleBinding
VERSION:  rbac.authorization.k8s.io/v1

DESCRIPTION:
     ClusterRoleBinding references a ClusterRole, but not contain it. It can
     reference a ClusterRole in the global namespace, and adds who information
     via Subject.

FIELDS:
   apiVersion   <string>
   kind <string>
   metadata     <Object>
   roleRef      <Object> -required-
     RoleRef can only reference a ClusterRole in the global namespace. If the
     RoleRef cannot be resolved, the Authorizer must return an error.
   subjects     <[]Object>
     Subjects holds references to the objects the role applies to.
```

RBAC可对Kubernetes资源进行增删改查CRUD(Create、Read、Update、Delete)操作的授权。常用资源（resource）如：Pods、ConfigMaps、Deployments、Nodes、Secrets、Namespaces、DaemonSet、StatefulSet和Service等

资源对象可能存在的操作(verb)如：create、get、delete、list、update、edit、watch、exec

### Role 和 ClusterRole

一个角色Role包含一组相关权限的规则。权限是累加的，不存在拒绝某操作的规则，角色可以用 Role 来定义到某个namespace上，或者用ClusterRole来定义到整个集群作用域，不受namespace的约束。

**Role示例**

授予对default命名空间中的 Pods 的读取权限：

```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""] # "" 指定核心 API 组
  resources: ["pods"]
  verbs: ["get", "watch", "list"]

```

**ClusterRole示例**

ClusterRole 可以授予的权限和 Role 相同，但是因为 ClusterRole 属于集群范围，所以它也可以授予以下访问权限：

- 集群范围资源（如 nodes）
- 非资源端点（如`/healthz`）
- 跨命名空间访问资源

```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
```

### RoleBinding和ClusterRoleBinding

角色绑定（RoleBinding）是将角色中定义的权限赋予一个用户或者一组用户。它包含若干主体`subjects`（users、groups或 serviceaccount）的列表和对这些主体所获得的角色引用`roleRef`

可以使用 RoleBinding 在指定的命名空间中执行授权，或者在集群范围的命名空间使用 ClusterRoleBinding 来执行授权

**RoleBinding示例**

```bash
apiVersion: rbac.authorization.k8s.io/v1
# 使得用户 "deemoprobe" 能够读取 "default" 命名空间中的 Pods
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: deemoprobe # 名称大小写敏感
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

roleRef 里的内容决定了实际创建绑定的方法。kind 可以是Role或ClusterRole。name 是引用的 Role 或 ClusterRole 的名称

**RoleBinding示例2**

RoleBinding 也可以引用 ClusterRole，这可以允许管理者在整个集群中定义一组通用的角色，然后在多个命名空间中重用它

```bash
apiVersion: rbac.authorization.k8s.io/v1
# 允许Jake用户在development命名空间中有读取 secrets 的权限
kind: RoleBinding
metadata:
  name: read-secrets
  namespace: development # 只授予development命名空间权限
subjects:
- kind: User
  name: Jake # 名称区分大小写
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io

```

**ClusterRoleBinding示例**

```bash
apiVersion: rbac.authorization.k8s.io/v1
# 允许manager组中的任何用户读取所有命名空间中secrets
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: manager # 名称区分大小写
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

当我们创建binding后，则不能修改binding所引用的Role或ClusterRole。尝试修改会导致验证错误，如果要改变binding的roleRef，可以删除该binding对象，创建一个新的。

### 聚合ClusterRole

```bash
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: monitoring
aggregationRule:
  clusterRoleSelectors: # 匹配所有具有标签的ClusterRole
  - matchLabels:
      rbac.example.com/aggregate-to-monitoring: "true"
rules: []

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: monitoring-endpoints
  labels:
    rbac.example.com/aggregate-to-monitoring: "true"
# 下面的规则将被添加进monitoring这个ClusterRole
rules:
- apiGroups: [""]
  resources: ["services", "endpoints", "pods"]
  verbs: ["get", "list", "watch"]
```

## RBAC实践

场景：赋予开发者dev在项目组1命名空间app-team1中资源Deployment、StatefulSet、DaemonSet、Pod的创建、读取、删除权限；赋予监控用户monitor查看pod和pod日志的权限。

### 准备工作

```bash
# 创建用户集中管理的namespace
[root@k8s-master01 ~]# kubectl create ns app-user
namespace/app-user created
# 创建项目app-team1的namespace
[root@k8s-master01 ~]# kubectl create ns app-team1
namespace/app-team1 created
# 创建项目app-team1开发者用户dev的serviceaccount
[root@k8s-master01 ~]# kubectl create sa dev -n app-user 
serviceaccount/dev created
# 创建集群监控用户monitor的serviceaccount
[root@k8s-master01 ~]# kubectl create sa monitor -n app-user
serviceaccount/monitor created

[root@k8s-master01 ~]# mkdir yamls/rbac/
[root@k8s-master01 ~]# cd yamls/rbac/
# 创建namespace只读clusterrole
[root@k8s-master01 rbac]# vim namespace-readonly.yaml 
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: namespace-readonly
rules:
- apiGroups:
  - ""
  resources:
  - namespaces
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - metrics.k8s.io
  resources:
  - pods
  verbs:
  - get
  - list
  - watch
# 创建app-team1 clusterrole
[root@k8s-master01 rbac]# cat app-team1-rbac.yaml 
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: app-team1-rabc
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - pods/exec
  - deployments
  - statefulsets
  - daemonsets
  verbs:
  - get
  - list
  - watch
  - create
  - delete
# 创建monitor clusterrole
[root@k8s-master01 rbac]# cat monitor-rbac.yaml 
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitor-rbac
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - pods/log
  verbs:
  - get
  - list
  - watch
[root@k8s-master01 rbac]# kubectl apply -f ./
clusterrole.rbac.authorization.k8s.io/app-team1-rabc created
clusterrole.rbac.authorization.k8s.io/monitor-rbac created
clusterrole.rbac.authorization.k8s.io/namespace-readonly created
# 授权两个用户namespace权限
[root@k8s-master01 rbac]# kubectl create clusterrolebinding namespace-readonly-dev --clusterrole=namespace-readonly --serviceaccount=app-user:dev
clusterrolebinding.rbac.authorization.k8s.io/namespace-readonly-dev created
[root@k8s-master01 rbac]# kubectl create clusterrolebinding namespace-readonly-monitor --clusterrole=namespace-readonly --serviceaccount=app-user:monitor
clusterrolebinding.rbac.authorization.k8s.io/namespace-readonly-monitor created
```

### 开发者授权

```bash
[root@k8s-master01 rbac]# kubectl create rolebinding -n app-team1 dev-rolebinding --serviceaccount=app-user:dev --clusterrole=app-team1-rabc
rolebinding.rbac.authorization.k8s.io/dev-rolebinding created
```

### 监控用户授权

```bash
[root@k8s-master01 rbac]# kubectl create rolebinding -n app-team1 monitor-rolebinding --serviceaccount=app-user:monitor --clusterrole=monitor-rbac
rolebinding.rbac.authorization.k8s.io/monitor-rolebinding created
```

### 登录验证

Dashboard端口为32601，TYPE为NodePort，使用任意一台集群主机访问：<https://IP:32601> 即可

[![1][1]][1]

[1]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220317133808.png

```bash
# 查看dashboard登陆端口
[root@k8s-master01 rbac]# kubectl get svc -n kubernetes-dashboard | grep kubernetes-dashboard
kubernetes-dashboard        NodePort    10.96.54.128    <none>        443:32601/TCP   10d

# 查看dev用户登陆token
[root@k8s-master01 rbac]# kubectl describe secret -n app-user dev-token-pmn64 
Name:         dev-token-pmn64
Namespace:    app-user
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: dev
              kubernetes.io/service-account.uid: 60810f7a-74af-4145-bf51-b1d6668ba0ca

Type:  kubernetes.io/service-account-token

Data
====
namespace:  8 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6ImFPeklobHBkNVRzZzZYVF9nbG5BMTgwOHdvMUNkV2FGbW1wdmUzZzdJRXcifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJhcHAtdXNlciIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkZXYtdG9rZW4tcG1uNjQiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGV2Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiNjA4MTBmN2EtNzRhZi00MTQ1LWJmNTEtYjFkNjY2OGJhMGNhIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmFwcC11c2VyOmRldiJ9.p0TMBvtG6T1vEDeWLELK77MOsLTZCNr2-kMN6e9NE38mDkoyIoh6BdM7DDVcW9QqhlCqwEYYNILsB25d6nGTawtwNrxq6m06io1diedfSOeSP2wZ4KpqVjoum0NxF3MjLgVhKNnpl_LayAuhxElEYpxdH42LyOPB_SYOdYXX-Vl4QUBus5dN3TwBPWhM0kvB_WE-3gq_HDebVcNlmZ_OKpcwxv5BFzsd8mn5vkS2loKnE10N5VKPAkotCSJlpv0HquhPX23aHjIevRcFqK3yzKAQ16oSZG8D5pzo0yaXgFf9U96GOU3YBrd-oCRHOiEzVfTw0hoAp1e5Mzsr_NDr4Q
ca.crt:     1411 bytes

# 查看monitor用户登陆token
[root@k8s-master01 rbac]# kubectl describe secret -n app-user monitor-token-kq7lf 
Name:         monitor-token-kq7lf
Namespace:    app-user
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: monitor
              kubernetes.io/service-account.uid: ddba4eee-c832-41d9-8c2c-cf1c34adcbe1

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1411 bytes
namespace:  8 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6ImFPeklobHBkNVRzZzZYVF9nbG5BMTgwOHdvMUNkV2FGbW1wdmUzZzdJRXcifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJhcHAtdXNlciIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJtb25pdG9yLXRva2VuLWtxN2xmIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6Im1vbml0b3IiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJkZGJhNGVlZS1jODMyLTQxZDktOGMyYy1jZjFjMzRhZGNiZTEiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6YXBwLXVzZXI6bW9uaXRvciJ9.u8hUnt6WtQI9KlZZyWj4Re-UL9ZcIFNxaiYxfI--IA3c9FROu6m6TSY0LTKgYb5xhcwG2a6fltrTCDDeMhh-pS02uixc153UYUST3OanBumpJ3kfsT4YpD7JtD6DojvMGc2cPjYDDe0HyW9oYao8RXUhlRaVdUi0Zb9sJ4r8_1VMOkgU_EWba1dgbvponYFBmPjdRnfQlxS8MJU2j0m1ZHI90Ru3NCiK9rUcZFlWSvxEc3rOp0L9aHXpUynrP5XXQ-DMJOY_hZvKmVsOFQ4nhI_UpX-OEPgLseuSzQm_HSUITGz3JenlyQY9TXML5_xTx-3hBQrk4dM2vlh7QR9Nng

# 在app-team1下创建一个deployment，去dashboard验证
[root@k8s-master01 rbac]# kubectl create deployment --image=nginx webapp -n app-team1 
deployment.apps/webapp created
```

注意：Secret资源`kubectl get secret name -oyaml`命令输出的是base64加密后的token，无法使用直接登陆Dashboard，`kubectl describe secret name`命令查看到的是解密后的token，可以直接用了认证登陆Dashboard。使用命令`kubectl -n app-user describe secret $(kubectl -n app-user get secret | grep dev | awk '{print $1}')`也可直接查看解密后的token，替换为自己想查看的namespace下secret名称即可。

- dev用户登陆后，可见提示`apps`组没有权限，还有其他一些没赋权的资源提示没权限

[![2][2]][2]

[2]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220317140633.png

```bash
# 添加apps组权限和replicationcontrollers资源后应用文件
[root@k8s-master01 rbac]# vim app-team1-rbac.yaml 
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: app-team1-rabc
rules:
- apiGroups:
  - ""
  - "apps"
  resources:
  - pods
  - pods/exec
  - deployments
  - statefulsets
  - daemonsets
  - replicationcontrollers
  verbs:
  - get
  - list
  - watch
  - create
  - delete
[root@k8s-master01 rbac]# kubectl apply -f app-team1-rbac.yaml 
clusterrole.rbac.authorization.k8s.io/app-team1-rabc configured
```

- 添加后刷新Dashboard可见dev用户界面之前的相关apps组合rc权限都消失了，主界面也展示了apps组已赋权的deployment资源，剩下的提示没权限是未赋权的资源

[![3][3]][3]

[3]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220317141204.png

- 点击一个pod进入查看日志，可见dev用户没有权限查看

[![4][4]][4]

[4]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220317141534.png

- 使用monitor用户登陆，可见是有权查看pod日志的

[![5][5]][5]

[5]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220317142026.png

至此，RBAC实践完成，其他规则只需要按需添加或删除即可。
