# Kubernetes安全机制之RBAC

Kubernetes通过一系列安全机制来确保集群的安全,包括API Server 的认证授权(Authentication/Authorization)/准入控制机制(Admission Control)/敏感信息的保护机制(Secret/Service_Account)等.

## 1. 认证「Authentication」

- **HTTP Token认证:** 通过一个Token来识别合法用户.HTTP Token的认证是用一个很长的特殊编码方式的并且难以被模仿的字符串来表达客户的一种方式.每一个Token对应一个用户名,存储在API Server能访问的文件中.当客户端发起API调用请求时,需要在HTTP Header里放入Token
- **HTTP Base认证:** 通过用户名+密码的方式认证.用户名:密码 用base64算法进行编码后的字符串放在HTTP Request中的Heather Authorization 域里发送给服务端,服务端收到后进行解码,获取用户名和密码
- **最严格的HTTPS证书认证:** 基于CA根证书签名的客户端身份认证方式

## 2. 授权「Authorization」

认证只是确认通信的双方都是可信的,可以相互通信.而授权是确定请求方有哪些资源的权限.API Server目前支持如下几种授权策略(通过API Server的启动参数 --authorization-mode 设置)

- AlwaysDeny：表示拒绝所有请求.仅用于测试
- AlwaysAllow：表示允许所有请求.如果有集群不需要授权流程,则可以采用该策略
- Node：节点授权是一种特殊用途的授权模式,专门授权由 kubelet 发出的 API 请求
- Webhook：是一种 HTTP 回调模式,允许使用远程 REST 端点管理授权
- ABAC(Attribute-Based Access Control)：基于属性的访问控制,表示使用用户配置的授权规则对用户请求进行匹配和控制
- RBAC(Role-Based Access Control)：基于角色的访问控制,默认使用该规则

## 3. RBAC授权模式

RBAC基于角色的访问控制,在Kubernetes 1.5 中引入,现为默认授权标准.相对其他访问控制方式,拥有如下优势：

- 对集群中的资源和非资源均拥有完整的覆盖
- 整个RBAC完全由几个API对象完成,同其他API对象一样,可以用kubectl或API进行操作
- 可以在运行时进行操作,无需重启API Server

```shell
[root@k8s-master apiserver]# kubectl get po --namespace=kube-system
NAME                                       READY   STATUS    RESTARTS   AGE
...
kube-apiserver-k8s-master                  1/1     Running   1          19h
# 可看到默认用的授权模式就是RBAC
[root@k8s-master apiserver]# kubectl describe po kube-apiserver-k8s-master --namespace=kube-system
Name:                 kube-apiserver-k8s-master
Namespace:            kube-system
Priority:             2000001000
...
IPs:
  IP:           192.168.43.10
Controlled By:  Node/k8s-master
Containers:
  kube-apiserver:
    ...
    Command:
      kube-apiserver
      --advertise-address=192.168.43.10
      --allow-privileged=true
      --authorization-mode=Node,RBAC
      --client-ca-file=/etc/kubernetes/pki/ca.crt
      --enable-admission-plugins=NodeRestriction,ResourceQuota
      --enable-bootstrap-token-auth=true
      ...
```

## 4. RBAC API资源对象

RBAC API 所声明的四种顶级类型[Role、ClusterRole、RoleBinding 和 ClusterRoleBinding].用户可以像与其他 API 资源交互一样,(通过 kubectl API 调用等方式)与这些资源交互.

![20210107104749](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20210107104749.png)

```shell
# 查看四个资源对象的apiVersion和描述信息
[root@k8s-master apiserver]# kubectl explain role
KIND:     Role
VERSION:  rbac.authorization.k8s.io/v1

DESCRIPTION:
     Role is a namespaced, logical grouping of PolicyRules that can be
     referenced as a unit by a RoleBinding.

FIELDS:
...
[root@k8s-master apiserver]# kubectl explain rolebinding
KIND:     RoleBinding
VERSION:  rbac.authorization.k8s.io/v1

DESCRIPTION:
     RoleBinding references a role, but does not contain it. It can reference a
     Role in the same namespace or a ClusterRole in the global namespace. It
     adds who information via Subjects and namespace information by which
     namespace it exists in. RoleBindings in a given namespace only have effect
     in that namespace.

FIELDS:
...
[root@k8s-master apiserver]# kubectl explain clusterrole
KIND:     ClusterRole
VERSION:  rbac.authorization.k8s.io/v1

DESCRIPTION:
     ClusterRole is a cluster level, logical grouping of PolicyRules that can be
     referenced as a unit by a RoleBinding or ClusterRoleBinding.

FIELDS:
...
[root@k8s-master apiserver]# kubectl explain clusterrolebinding
KIND:     ClusterRoleBinding
VERSION:  rbac.authorization.k8s.io/v1

DESCRIPTION:
     ClusterRoleBinding references a ClusterRole, but not contain it. It can
     reference a ClusterRole in the global namespace, and adds who information
     via Subject.

FIELDS:
...
```

Kubernetes可对API对象执行增删改查CRUD(Create、Read、Update、Delete)操作. 常用资源如下:

- Pods
- ConfigMaps
- Deployments
- Nodes
- Secrets
- Namespaces

资源对象可能存在的操作(verb)有如下:

- create
- get
- delete
- list
- update
- edit
- watch
- exec

### 4.1. Role 和 ClusterRole

在 RBAC API 中,一个角色包含一组相关权限的规则.权限是纯粹累加的(不存在拒绝某操作的规则),即只能给权限累加,不存在给了XX权限,然后去掉XX01权限的情况.角色可以用 Role 来定义到某个命名空间(namespace)上, 或者用 ClusterRole 来定义到整个集群作用域(所有namespace).

一个 Role 只可以用来对某一命名空间中的资源赋予访问权限.

#### 4.1.1. Role示例

定义到名称为 “default” 的命名空间,可以用来授予对该命名空间中的 Pods 的读取权限：

```shell
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

#### 4.1.2. ClusterRole示例

ClusterRole 可以授予的权限和 Role 相同,但是因为 ClusterRole 属于集群范围,所以它也可以授予以下访问权限：

- 集群范围资源 (比如 nodes访问)
- 非资源端点(比如 “/healthz” 访问)
- 跨命名空间访问的有名称空间作用域的资源(如 Pods),比如运行命令kubectl get pods --all-namespaces 时需要此能力

可用来对某特定命名空间下的 Secrets 的读取操作授权,或者跨所有名称空间执行授权(取决于它是如何绑定的)：

```shell
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
  # 此处的 "namespace" 被省略掉是因为 ClusterRoles 是没有命名空间的.
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
```

### 4.2. RoleBinding 和 ClusterRoleBinding

角色绑定(RoleBinding)是将角色中定义的权限赋予一个用户或者一组用户. 它包含若干主体[subjects](users、groups或 service accounts)的列表和对这些主体所获得的角色引用.

可以使用 RoleBinding 在指定的命名空间中执行授权,或者在集群范围的命名空间使用 ClusterRoleBinding 来执行授权.

一个 RoleBinding 可以引用同一的命名空间中的 Role.

#### 4.2.1. RoleBinding示例

将 “pod-reader” 角色授予在 “default” 命名空间中的用户 “deemo”; 这样,用户 “deemo” 就具有了读取 “default” 命名空间中 pods 的权限.

在下面的例子中,角色绑定使用 roleRef 将用户 “deemo” 绑定到前文创建的角色 Role,其名称是 pod-reader.

```shell
apiVersion: rbac.authorization.k8s.io/v1
# 此角色绑定,使得用户 "deemo" 能够读取 "default" 命名空间中的 Pods
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: deemo # 名称大小写敏感
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role #this must be Role or ClusterRole
  name: pod-reader # 这里的名称必须与你想要绑定的 Role 或 ClusterRole 名称一致
  apiGroup: rbac.authorization.k8s.io
```

roleRef 里的内容决定了实际创建绑定的方法.kind 可以是 Role 或 ClusterRole,name 是你要引用的 Role 或 ClusterRole 的名称.

RoleBinding 也可以引用 ClusterRole,这可以允许管理者在 整个集群中定义一组通用的角色,然后在多个命名空间中重用它们.

#### 4.2.2. RoleBinding示例2

下面的例子,RoleBinding 引用的是 ClusterRole, “Jake” (subjects区分大小写)将只可以读取在”development” 名称空间( RoleBinding 的命名空间)中的”secrets” .

```shell
apiVersion: rbac.authorization.k8s.io/v1
# 这个角色绑定允许 "Jake" 用户在 "development" 命名空间中有读取 secrets 的权限. 
kind: RoleBinding
metadata:
  name: read-secrets
  namespace: development # 这里只授予 "development" 命名空间的权限.
subjects:
- kind: User
  name: Jake # 名称区分大小写
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io

```

最后,ClusterRoleBinding 可用来在集群级别并对所有命名空间执行授权.

#### 4.2.3. ClusterRoleBinding示例

```shell
apiVersion: rbac.authorization.k8s.io/v1
# 这个集群角色绑定允许 "manager" 组中的任何用户读取任意命名空间中 "secrets".
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

当我们创建binding后,则不能修改binding所引用的Role或ClusterRole.尝试修改会导致验证错误;如果要改变binding的roleRef,那么应该删除该binding对象并且创建一个新的用来替换原来的.

## 5. Referring to resources[资源引用]

Kubernetes集群内一些资源一般以其名称字符串来表示,这些字符串一般会在API的URL地址中出现;同时某些资源也会包含子资源,例如pod的logs资源就属于pods的子资源,API中URL样例如下：

```shell
GET /api/v1/namespaces/{namespace}/pods/{name}/log
```

在这种情况下,”pods” 是有名称空间的资源,而 “log” 是 pods 的子资源.在 RBAC 角色中,使用”/“分隔资源和子资源.

允许一个主体(subject)要同时读取 pods 和 pod logs,你可以这么写：

```shell
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-and-pod-logs-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list"]

```

对于某些请求,也可以通过 resourceNames 列表按名称引用资源.

例如：在指定时,可以将请求类型限制到资源的单个实例.限制只可以 “get” 和 “update” 到单个configmap,则可以这么写：

```shell
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: configmap-updater
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["my-configmap"]
  verbs: ["update", "get"]

```

需要注意的是,create 请求不能被 resourceName 限制,因为在鉴权时还不知道对象名称. 另一个例外是 deletecollection.

## 6. Referring to subjects[主体引用]

RoleBinding或ClusterRoleBinding绑定一个role到主体(subjects).主体(subjects)可以是groups,users或ServiceAccounts.

Kubernetes将用户名表示为字符串.这些可以是：普通名称,比如“deemo” ;邮件风格的名字,比如“bob@example.com” ;或表示为字符串的数字用户id.

注意：前缀 `system:` 是保留给Kubernetes系统使用的,因此应该确保不会出现名称以system: 开头的用户或组.除了这个特殊的前缀,RBAC授权系统不要求用户名使用任何格式.

ServiceAccounts具有前缀为system:serviceaccount: 的名称,属于具有前缀为system:serviceaccounts:的名称的组.

### 6.1. RoleBinding的示例

下面的示例只是展示 RoleBinding 中 subjects 的部分.

用户的名称为 “deemo@example.com” ：

```shell
subjects:
- kind: User
  name: "deemo@example.com"
  apiGroup: rbac.authorization.k8s.io

```

组的名称为 “frontend-admins” ：

```shell
subjects:
- kind: Group
  name: "frontend-admins"
  apiGroup: rbac.authorization.k8s.io

```

默认service account在 kube-system 命名空间中：

```shell
subjects:
- kind: ServiceAccount
  name: default
  namespace: kube-system

```

在名称为 “qa” 命名空间中所有的服务账号：

```shell
subjects:
- kind: Group
  name: system:serviceaccounts:qa
  apiGroup: rbac.authorization.k8s.io

```

在任意名称空间的所有service accounts：

```shell
subjects:
- kind: Group
  name: system:serviceaccounts
  apiGroup: rbac.authorization.k8s.io

```

所有认证过的用户(版本 1.5+)：

```shell
subjects:
- kind: Group
  name: system:authenticated
  apiGroup: rbac.authorization.k8s.io
```

所有未认证的用户(版本 1.5+)：

```shell
subjects:
- kind: Group
  name: system:unauthenticated
  apiGroup: rbac.authorization.k8s.io
```

所有用户 (版本 1.5+)：

```shell
subjects:
- kind: Group
  name: system:authenticated
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: system:unauthenticated
  apiGroup: rbac.authorization.k8s.io

```

## 7. 准入控制

准入控制是API Server的插件集合,通过添加不同的插件,实现额外的准入控制规则.甚至于API Server的一些主要的功能都需要通过Admission Controllers实现,比如：ServiceAccount.

一般1.17+版本以下插件会默认启用：

```shell
NamespaceLifecycle, LimitRanger, ServiceAccount, 
TaintNodesByCondition, Priority, DefaultTolerationSeconds, 
DefaultStorageClass, StorageObjectInUseProtection, PersistentVolumeClaimResize, 
MutatingAdmissionWebhook, ValidatingAdmissionWebhook, RuntimeClass, ResourceQuota
```

启用/关闭插件(--enable-admission-plugins/--disable-admission-plugins)

```shell
# 方式一: 命令编辑apiserver Pod
[root@k8s-master apiserver]# kubectl edit po kube-apiserver-k8s-master --namespace=kube-system
...
spec:
  containers:
  - command:
    - kube-apiserver
    ...
    - --enable-admission-plugins=NodeRestriction,ResourceQuota
    ...
# 方式二: 直接编辑配置文件, 文件路径默认是/etc/kubernetes/manifests/kube-apiserver.yaml
[root@k8s-master apiserver]# vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

### 7.1. 部分插件功能

#### 7.1.1. NamespaceLifecycle

该准入控制器禁止在一个正在被终止的 Namespace 中创建新对象,并确保使用不存在的 Namespace 的请求被拒绝.该准入控制器还会禁止删除三个系统保留的命名空间,即 default、kube-system 和 kube-public.

删除 Namespace 会触发删除该命名空间中所有对象(pod、services 等)的一系列操作.为了确保这个过程的完整性,我们强烈建议启用这个准入控制器.

#### 7.1.2. LimitRanger

该准入控制器会观察传入的请求,并确保它不会违反 Namespace 中 LimitRange 对象枚举的任何约束.

#### 7.1.3. ServiceAccount

该准入控制器实现了 serviceAccounts 的自动化. 如果打算使用 Kubernetes 的 ServiceAccount 对象,强烈建议您使用这个准入控制器.

#### 7.1.4. ResourceQuota

该准入控制器会监测传入的请求,并确保它不违反任何一个 Namespace 中的 ResourceQuota 对象中枚举出来的约束.
