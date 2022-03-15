# Kubernetes配置管理之Secret

2021-0722

## 概念

Secret 是一种包含少量敏感信息例如密码、令牌或密钥的对象。使用 Secret 意味着可以独立于应用之外存储敏感数据。由于创建 Secret 可以独立于使用它们的 Pod， 因此在创建、查看和编辑Pod的工作流程中暴露Secret的风险较小。Kubernetes和在集群中运行的应用程序也可以对 Secret 采取额外的预防措施，例如避免将机密数据写入非易失性存储。

Secret 类似于 ConfigMap 但专门用于保存机密数据。用户可以创建 secret,同时系统也创建了一些 secret。

Secret解决了密码、token、秘钥等敏感数据的配置问题，而不需要把这些敏感数据暴露到镜像或者Pod Spec中。Secret可以以Volume或者环境变量的方式使用.

> 注意：默认情况下，Kubernetes Secret未加密地存储在 API 服务器的底层数据存储（etcd）中。任何拥有 API 访问权限的人都可以检索或修改 Secret，任何有权访问 etcd 的人也可以。此外，任何有权限在命名空间中创建Pod的人都可以使用该访问权限读取该命名空间中的任何Secret；这包括间接访问，例如创建 Deployment 的能力。为了安全地使用 Secret，至少要为secret配置：1.[Secret启用静态加密](https://kubernetes.io/zh/docs/tasks/administer-cluster/encrypt-data/)；2.启用或配置RBAC规则来限制读取Secret数据（包括通过间接方式）；3.在适当的情况下，还可以使用RBAC等机制来限制允许哪些主体创建新Secret或更新现有Secret

Secret内置的`type`类型:

- Opaque：默认类型，用户定义的任意数据
- kubernetes.io/service-account-token：服务账号令牌
- kubernetes.io/dockercfg	~/.dockercfg：文件的序列化形式
- kubernetes.io/dockerconfigjson：~/.docker/config.json 文件的序列化形式
- kubernetes.io/basic-auth：用于基本身份认证的凭据
- kubernetes.io/ssh-auth：用于 SSH 身份认证的凭据
- kubernetes.io/tls：用于 TLS 客户端或者服务器端的数据
- bootstrap.kubernetes.io/token：启动引导令牌数据

## 类型实例

### Opaque Secret

```bash
[root@k8s-master01 ~]# mkdir -p yamls/secret
[root@k8s-master01 ~]# cd yamls/secret/
# generic子命令表明创建的是普通类型的secret，即默认为opaque类型
[root@k8s-master01 secret]# kubectl create secret generic opaque-secret
secret/opaque-secret created
[root@k8s-master01 secret]# kubectl get secrets 
NAME                  TYPE                                  DATA   AGE
opaque-secret         Opaque                                0      5s
[root@k8s-master01 secret]# kubectl describe secrets opaque-secret 
Name:         opaque-secret
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data # DATA==0 没有数据
====

# 创建用户凭证，去除换行符，否则base64加密也会包含换行符
[root@k8s-master01 secret]# echo -n 'admin' > ./username.txt
[root@k8s-master01 secret]# echo -n 'fag734ubjtb28821hhna' > ./password.txt
# 默认文件名为秘钥名称，也可以指定--from-file=username=./username.txt
[root@k8s-master01 secret]# kubectl create secret generic user-passwd --from-file=./username.txt --from-file=./password.txt 
secret/user-passwd created
[root@k8s-master01 secret]# kubectl get secrets user-passwd 
NAME          TYPE     DATA   AGE
user-passwd   Opaque   2      9s
[root@k8s-master01 secret]# kubectl describe secrets user-passwd 
Name:         user-passwd
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password.txt:  20 bytes
username.txt:  5 bytes

# 也可以直接使用参数方式创建凭证，密码如果有特殊字符（例如：$，\，*，= 和 !），需要用单引号转义为字符串
[root@k8s-master01 secret]# kubectl create secret generic user-password --from-literal=username=admin --from-literal=password='ninda$\ad*^'
secret/user-password created
[root@k8s-master01 secret]# kubectl describe secrets user-password 
Name:         user-password
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password:  11 bytes
username:  5 bytes

# 获取secret加密内容
[root@k8s-master01 secret]# kubectl get secrets user-password -o jsonpath='{.data}'
{"password":"bmluZGEkXGFkKl4=","username":"YWRtaW4="}
# 解码，base64 --decode，简写base64 -d，这里没有换行符，所以效果如下
[root@k8s-master01 secret]# echo 'YWRtaW4=' | base64 -d
admin[root@k8s-master01 secret]# echo 'bmluZGEkXGFkKl4=' | base64 -d
ninda$\ad*^

# 删除secret
[root@k8s-master01 secret]# kubectl delete secrets user-password 
secret "user-password" deleted
```

### service-account

默认每个namespace都会生成一个default服务账户，Pod默认使用这个账户，如果Pod要使用指定的serviceaccount账户需要指定`spec.serviceAccountName`字段。证书令牌会自动挂载到Pod的 `/run/secrets/kubernetes.io/serviceaccount`目录中

```bash
[root@k8s-master01 secret]# kubectl get sa
NAME          SECRETS   AGE
default       1         9d
[root@k8s-master01 secret]# kubectl get sa default -oyaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2022-03-06T06:24:39Z"
  name: default
  namespace: default
  resourceVersion: "547"
  uid: 24a36bad-475a-4086-90f8-f030711edfda
secrets:
- name: default-token-wnp8m
[root@k8s-master01 secret]# kubectl get secrets default-token-wnp8m -oyaml
apiVersion: v1
data:
  ca.crt: 
  已省略
  namespace: ZGVmYXVsdA==
  token: 
  已省略
kind: Secret
metadata:
  annotations:
    kubernetes.io/service-account.name: default
    kubernetes.io/service-account.uid: 24a36bad-475a-4086-90f8-f030711edfda
  creationTimestamp: "2022-03-06T06:24:39Z"
  name: default-token-wnp8m
  namespace: default
  resourceVersion: "546"
  uid: a0c7b3cd-aed8-4e5f-b3c7-ee5d1eae90b3
type: kubernetes.io/service-account-token

# 随便找一个pod，查看证书令牌的挂载目录
[root@k8s-master01 secret]# kubectl get po nginx-85b98978db-sfzmb 
NAME                     READY   STATUS    RESTARTS      AGE
nginx-85b98978db-sfzmb   1/1     Running   5 (71m ago)   2d19h
[root@k8s-master01 secret]# kubectl exec -it nginx-85b98978db-sfzmb -- ls -al /run/secrets/kubernetes.io/serviceaccount
total 0
drwxrwxrwt 3 root root 140 Mar 15 06:37 .
drwxr-xr-x 3 root root  28 Mar 15 05:50 ..
drwxr-xr-x 2 root root 100 Mar 15 06:37 ..2022_03_15_06_37_46.2287599106
lrwxrwxrwx 1 root root  32 Mar 15 06:37 ..data -> ..2022_03_15_06_37_46.2287599106
lrwxrwxrwx 1 root root  13 Mar 15 05:49 ca.crt -> ..data/ca.crt
lrwxrwxrwx 1 root root  16 Mar 15 05:49 namespace -> ..data/namespace
lrwxrwxrwx 1 root root  12 Mar 15 05:49 token -> ..data/token

# 创建serviceaccount资源，简写sa
[root@k8s-master01 secret]# kubectl create serviceaccount web-account
serviceaccount/web-account created
[root@k8s-master01 secret]# kubectl describe sa web-account 
Name:                web-account
Namespace:           default
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   web-account-token-96q8n
Tokens:              web-account-token-96q8n
Events:              <none>
# 可查看到默认为指定的sa配置了一个secret：web-account-token-96q8n
[root@k8s-master01 secret]# kubectl get sa web-account -oyaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2022-03-15T06:29:59Z"
  name: web-account
  namespace: default
  resourceVersion: "181175"
  uid: dfcbd0e7-8ab4-4f8a-b897-32cb294c9eee
secrets:
- name: web-account-token-96q8n
# 查看secret详情
[root@k8s-master01 secret]# kubectl get secrets web-account-token-96q8n -oyaml
apiVersion: v1
data:
  ca.crt: 
  ca证书内容已省略
  namespace: ZGVmYXVsdA== # 加密后的namespace名称
  token: 
  token串内容已省略
kind: Secret
metadata:
  annotations:
    kubernetes.io/service-account.name: web-account
    kubernetes.io/service-account.uid: dfcbd0e7-8ab4-4f8a-b897-32cb294c9eee
  creationTimestamp: "2022-03-15T06:29:59Z"
  name: web-account-token-96q8n
  namespace: default
  resourceVersion: "181174"
  uid: 2c393434-8793-4dc1-b314-2f3185bb28ca
type: kubernetes.io/service-account-token
# 解码看一下namespace名称
[root@k8s-master01 secret]# echo 'ZGVmYXVsdA==' | base64 -d
default
```

### dockerconfigjson

首先需要有一个镜像仓库, 比如Docker Hub或者 Harbor搭建一个镜像仓库

通过以下方式去注册一个docker secret认证, 方可拉取私有镜像

kubectl create secret docker-registry myregistrysecret --docker-server='IP:Port' --docker-username='name' --docker-password='passwd'

```bash
# 比如镜像仓库地址为172.16.1.1:5000, 用户名admin, 密码Test123456, 私有镜像为myapp
[root@k8s-master01 secret]# kubectl create secret docker-registry myregistrysecret --docker-server='172.16.1.1:5000' --docker-username='admin' --docker-password='Test123456' 
secret/myregistrysecret created
[root@k8s-master01 secret]# kubectl get secret 
NAME                  TYPE                                  DATA   AGE
myregistrysecret      kubernetes.io/dockerconfigjson        1      8s
[root@k8s-master01 secret]# kubectl describe secret myregistrysecret
Name:         myregistrysecret
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  kubernetes.io/dockerconfigjson

Data
====
.dockerconfigjson:  112 bytes
# 创建pod, 拉取私有镜像
[root@k8s-master01 secret]# cat pod_secret_registry.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: pod-secret-registry
spec:
  containers:
  - name: myapp
    image: 172.16.1.1:5000/myapp
  imagePullSecrets:
  - name: myregistrysecret
[root@k8s-master01 secret]# kubectl apply -f pod_secret_registry.yaml 
pod/pod-secret-registry created
[root@k8s-master01 secret]# kubectl describe pod pod-secret-registry
Name:         pod-secret-registry
Namespace:    default
...
Volumes:
  default-token-v48g4:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-t65f5
    Optional:    false
QoS Class:       BestEffort
...
Events:
  Type    Reason     Age   From                 Message
  ----    ------     ----  ----                 -------
  Normal  Scheduled  22s   default-scheduler    Successfully assigned default/pod-secret-registry to k8s-node1
  Normal  Pulling    22s   kubelet, k8s-node1  Pulling image "172.16.1.1:5000/myapp"
  Normal  Pulled     22s   kubelet, k8s-node1  Successfully pulled image "172.16.1.1:5000/myapp"
  Normal  Created    22s   kubelet, k8s-node1  Created container myapp
  Normal  Started    21s   kubelet, k8s-node1  Started container myapp
```

### basic-auth

`kubernetes.io/basic-auth`类型用来存放用于基本身份认证所需的凭据信息。使用这种Secret类型时，Secret的data字段必须包含以下两个键：

- username: 用于身份认证的用户名；
- password: 用于身份认证的密码或令牌。

以上两个键的键值都是 base64 编码的字符串。当然也可以在创建Secret时使用`stringData`字段来提供明文形式的内容。提供基本身份认证类型的 Secret 仅仅是出于用户方便性考虑。

```bash
# 明文内容
apiVersion: v1
kind: Secret
metadata:
  name: secret-basic-auth
type: kubernetes.io/basic-auth
stringData:
  username: admin
  password: t0p-Secret
```

### ssh-auth

Kubernetes 所提供的内置类型`kubernetes.io/ssh-auth`用来存放SSH身份认证中所需要的凭据。使用这种Secret类型时，你就必须在其data（或stringData）字段中提供一个`ssh-privatekey`键值对，作为要使用的SSH凭据。

```bash
apiVersion: v1
kind: Secret
metadata:
  name: secret-ssh-auth
type: kubernetes.io/ssh-auth
data:
  ssh-privatekey: |
          MIIEpQIBAAKCAQEAulqb/Y...已省略
```

> 注意： SSH 私钥自身无法建立 SSH 客户端与服务器端之间的可信连接。 需要其它方式来建立这种信任关系，以缓解“中间人（Man In The Middle）” 攻击，例如向 ConfigMap 中添加一个 known_hosts 文件。

### tls

Kubernetes 提供一种内置的`kubernetes.io/tls` Secret 类型，用来存放证书 及其相关密钥（通常用在 TLS 场合）。 此类数据主要提供给 Ingress 资源，用以终结 TLS 链接，不过也可以用于其他 资源或者负载。当使用此类型的 Secret 时，Secret 配置中的 data （或 stringData）字段必须包含`tls.key`和`tls.crt`主键，尽管 API 服务器 实际上并不会对每个键的取值作进一步的合法性检查。

这里的公钥/私钥对都必须事先已存在。用于`--cert`的公钥证书必须是 .PEM 编码的 （Base64 编码的 DER 格式），且与`--key`所给定的私钥匹配。 私钥必须是通常所说的 PEM 私钥格式，且未加密。对这两个文件而言，PEM 格式数据 的第一行和最后一行（例如，证书所对应的`--------BEGIN CERTIFICATE-----`和`-------END CERTIFICATE----`）都不会包含在其中。

```bash
apiVersion: v1
kind: Secret
metadata:
  name: secret-tls
type: kubernetes.io/tls
data:
  tls.crt: |
        MIIC2DCCAcCgAwIBAgIBATANBgkqh...已省略
  tls.key: |
        MIIEpgIBAAKCAQEA7yn3bRHQ5FHMQ...已省略

# 也可以直接使用命令指定文件位置
kubectl create secret tls my-tls-secret \
  --cert=path/to/cert/file \
  --key=path/to/key/file
```

## 不可变的secret

类似不可变的ConfigMap，一经创建无法后期修改，除非删除重新创建。

```bash
apiVersion: v1
kind: Secret
metadata:
  ...
data:
  ...
immutable: true
```

## 挂载secret

```bash
[root@k8s-master01 secret]# kubectl get secrets 
NAME                      TYPE                                  DATA   AGE
default-token-wnp8m       kubernetes.io/service-account-token   3      9d
opaque-secret             Opaque                                0      95m
user-passwd               Opaque                                2      72m
web-account-token-96q8n   kubernetes.io/service-account-token   3      56m
# 创建yaml
[root@k8s-master01 secret]# vim pod-secret-vol.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-secret-volume
spec:
  containers:
  - name: myapp
    image: registry.cn-beijing.aliyuncs.com/google_registry/myapp:v1
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secret
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: user-passwd # 使用这个已创建的secret
[root@k8s-master01 secret]# kubectl apply -f pod_secret_volume.yaml 
pod/pod-secret-volume created
[root@k8s-master01 secret]# kubectl get po pod-secret-volume 
NAME                READY   STATUS    RESTARTS   AGE
pod-secret-volume   1/1     Running   0          26s
[root@k8s-master01 secret]# kubectl exec -it pod-secret-volume -- cat /etc/secret/username.txt
admin[root@k8s-master01 secret]# kubectl exec -it pod-secret-volume -- cat /etc/secret/password.txt
fag734ubjtb28821hhna
```

由上可见,在pod中的secret信息实际已经被解密。
