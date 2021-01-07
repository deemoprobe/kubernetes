# Pod 安全策略

PodSecurityPolicy 类型的对象能够控制,是否可以向 Pod 发送请求,该 Pod 能够影响被应用到 Pod 和容器的 SecurityContext. 查看 Pod 安全策略建议 获取更多信息.

## 什么是 Pod 安全策略

Pod 安全策略 是集群级别的资源,它能够控制 Pod 运行的行为,以及它具有访问什么的能力. PodSecurityPolicy对象定义了一组条件,指示 Pod 必须按系统所能接受的顺序运行. 它们允许管理员控制如下方面:

| 控制面                           | 字段名称                 |
| -------------------------------- | ------------------------ |
| 已授权容器的运行                 | privileged               |
| 为容器添加默认的一组能力         | defaultAddCapabilities   |
| 为容器去掉某些能力               | requiredDropCapabilities |
| 容器能够请求添加某些能力         | allowedCapabilities      |
| 控制卷类型的使用                 | volumes                  |
| 主机网络的使用                   | hostNetwork              |
| 主机端口的使用                   | hostPorts                |
| 主机 PID namespace 的使用        | hostPID                  |
| 主机 IPC namespace 的使用        | hostIPC                  |
| 主机路径的使用                   | allowedHostPaths         |
| 容器的 SELinux 上下文            | seLinux                  |
| 用户 ID                          | runAsUser                |
| 配置允许的补充组                 | supplementalGroups       |
| 分配拥有 Pod 数据卷的 FSGroup    | fsGroup                  |
| 必须使用一个只读的 root 文件系统 | readOnlyRootFilesystem   |

Pod 安全策略 由设置和策略组成,它们能够控制 Pod 访问的安全特征.这些设置分为如下三类:

- 基于布尔值控制:这种类型的字段默认为最严格限制的值.
- 基于被允许的值集合控制:这种类型的字段会与这组值进行对比,以确认值被允许.
- 基于策略控制:设置项通过一种策略提供的机制来生成该值,这种机制能够确保指定的值落在被允许的这组值中.

### RunAsUser

MustRunAs - 必须配置一个 range.使用该范围内的第一个值作为默认值.验证是否不在配置的该范围内.
MustRunAsNonRoot - 要求提交的 Pod 具有非零 runAsUser 值,或在镜像中定义了 USER 环境变量.不提供默认值.
RunAsAny - 没有提供默认值.允许指定任何 runAsUser .

### SELinux

MustRunAs - 如果没有使用预分配的值,必须配置 seLinuxOptions.默认使用 seLinuxOptions.验证 seLinuxOptions.
RunAsAny - 没有提供默认值.允许任意指定的 seLinuxOptions ID.

### SupplementalGroups

MustRunAs - 至少需要指定一个范围.默认使用第一个范围的最小值.验证所有范围的值.
RunAsAny - 没有提供默认值.允许任意指定的 supplementalGroups ID.

### FSGroup

MustRunAs - 至少需要指定一个范围.默认使用第一个范围的最小值.验证在第一个范围内的第一个 ID.
RunAsAny - 没有提供默认值.允许任意指定的 fsGroup ID.

### 控制卷

通过设置 PSP 卷字段,能够控制具体卷类型的使用.当创建一个卷的时候,与该字段相关的已定义卷可以允许设置如下值:

- azureFile
- azureDisk
- flocker
- flexVolume
- hostPath
- emptyDir
- gcePersistentDisk
- awsElasticBlockStore
- gitRepo
- secret
- nfs
- iscsi
- glusterfs
- persistentVolumeClaim
- rbd
- cinder
- cephFS
- downwardAPI
- fc
- configMap
- vsphereVolume
- quobyte
- photonPersistentDisk
- projected
- portworxVolume
- scaleIO
- storageos
- "*" (allow all volumes)

对新的 PSP,推荐允许的卷的最小集合包括:configMap、downwardAPI、emptyDir、persistentVolumeClaim、secret 和 projected.

### 主机网络

HostPorts, 默认为 empty.HostPortRange 列表通过 min(包含) and max(包含) 来定义,指定了被允许的主机端口.

### 允许的主机路径

AllowedHostPaths 是一个被允许的主机路径前缀的白名单.空值表示所有的主机路径都可以使用.

## 许可

包含 PodSecurityPolicy 的 许可控制,允许控制集群资源的创建和修改,基于这些资源在集群范围内被许可的能力.

许可使用如下的方式为 Pod 创建最终的安全上下文:

- 检索所有可用的 PSP.
- 生成在请求中没有指定的安全上下文设置的字段值.
- 基于可用的策略,验证最终的设置.

如果某个策略能够匹配上,该 Pod 就被接受.如果请求与 PSP 不匹配,则 Pod 被拒绝.

Pod 必须基于 PSP 验证每个字段.

## 创建 Pod 安全策略

下面是一个 Pod 安全策略的例子,所有字段的设置都被允许:

```shell
# 查看PodSecurityPolicy apiVersion和描述信息, 简写为psp
[root@k8s-master apiserver]# kubectl explain psp
KIND:     PodSecurityPolicy
VERSION:  policy/v1beta1

DESCRIPTION:
     PodSecurityPolicy governs the ability to make requests that affect the
     Security Context that will be applied to a pod and container.

FIELDS:
...
# 创建PSP
[root@k8s-master apiserver]# vi psp.yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: permissive
spec:
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  runAsUser:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  hostPorts:
  - min: 8000
    max: 8080
  volumes:
  - '*'
[root@k8s-master apiserver]# kubectl apply -f psp.yaml 
podsecuritypolicy.policy/permissive created
```

## 获取 Pod 安全策略列表

获取已存在策略列表,使用 kubectl get:

```shell
[root@k8s-master apiserver]# kubectl get psp
NAME         PRIV    CAPS   SELINUX    RUNASUSER   FSGROUP    SUPGROUP   READONLYROOTFS   VOLUMES
permissive   false          RunAsAny   RunAsAny    RunAsAny   RunAsAny   false            *
[root@k8s-master apiserver]# kubectl describe psp permissive
Name:         permissive
Namespace:    
Labels:       <none>
Annotations:  <none>
API Version:  policy/v1beta1
Kind:         PodSecurityPolicy
Metadata:
  Creation Timestamp:  2021-01-07T04:21:01Z
  Managed Fields:
    API Version:  policy/v1beta1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .:
          f:kubectl.kubernetes.io/last-applied-configuration:
      f:spec:
        f:allowPrivilegeEscalation:
        f:fsGroup:
          f:rule:
        f:hostPorts:
        f:runAsUser:
          f:rule:
        f:seLinux:
          f:rule:
        f:supplementalGroups:
          f:rule:
        f:volumes:
    Manager:         kubectl-client-side-apply
    Operation:       Update
    Time:            2021-01-07T04:21:01Z
  Resource Version:  681980
  Self Link:         /apis/policy/v1beta1/podsecuritypolicies/permissive
  UID:               c92a661a-d3c7-4112-bd52-8c75fc8f8b37
Spec:
  Allow Privilege Escalation:  true
  Fs Group:
    Rule:  RunAsAny
  Host Ports:
    Max:  8080
    Min:  8000
  Run As User:
    Rule:  RunAsAny
  Se Linux:
    Rule:  RunAsAny
  Supplemental Groups:
    Rule:  RunAsAny
  Volumes:
    *
...
```

## 修改 Pod 安全策略

通过交互方式修改策略,使用 kubectl edit:

```shell
# 打开一个默认文本编辑器,修改策略
[root@k8s-master apiserver]# kubectl edit psp permissive
```

## 删除 Pod 安全策略

一旦不再需要一个策略,很容易通过 kubectl 删除它:

```shell
[root@k8s-master apiserver]# kubectl delete psp permissive
podsecuritypolicy.policy "permissive" deleted
```

## 使用 Pod 安全策略

### 在APIServer中启用PSP

```shell
[root@k8s-master apiserver]# vi /etc/kubernetes/manifests/kube-apiserver.yaml
apiVersion: v1
kind: Pod
metadata:
  ...
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    ...
    - --enable-admission-plugins=...,PodSecurityPolicy
    ...
```

### 授权PSP

通过RBAC授权, 类似于下面的配置

```shell
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: <Role 名称>
rules:
- apiGroups: ['policy']
  resources: ['podsecuritypolicies']
  verbs:     ['use']
  resourceNames:
  - <要授权的策略列表>
```

然后绑定即可

```shell
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: <绑定名称>
roleRef:
  kind: ClusterRole
  name: <角色名称>
  apiGroup: rbac.authorization.k8s.io
subjects:
# 授权特定的服务账号
- kind: ServiceAccount
  name: <要授权的服务账号名称>
  namespace: <authorized pod namespace>
# 授权特定的用户（不建议这样操作）
- kind: User
  apiGroup: rbac.authorization.k8s.io
  name: <要授权的用户名>
```
