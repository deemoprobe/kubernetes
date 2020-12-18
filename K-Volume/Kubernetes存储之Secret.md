# Kubernetes存储之Secret

Secret解决了密码、token、秘钥等敏感数据的配置问题,而不需要把这些敏感数据暴露到镜像或者Pod Spec中.Secret可以以Volume或者环境变量的方式使用.

用户可以创建 secret,同时系统也创建了一些 secret.

要使用 secret,pod 需要引用 secret.Pod 可以用两种方式使用 secret: 作为 volume 中的文件被挂载到 pod 中的一个或者多个容器里,或者当 kubelet 为 pod 拉取镜像时使用.

Secret类型:

- Service Account: 用来访问Kubernetes API,由Kubernetes自动创建,并且会自动挂载到Pod的 `/run/secrets/kubernetes.io/serviceaccount` 目录中
- Opaque: base64编码格式的Secret,用来存储密码、秘钥等
- kubernetes.io/dockerconfigjson: 用来存储私有docker registry的认证信息

## 1. Service Account

通过kube-proxy查看

```shell
[root@k8s-master volume]# kubectl get pod -A | grep 'kube-proxy' 
kube-system   kube-proxy-2ld6t                           1/1     Running            18         34d
kube-system   kube-proxy-bx9jg                           1/1     Running            17         34d
[root@k8s-master volume]# kubectl exec -it -n kube-system kube-proxy-2ld6t -- /bin/sh
# ls -l /run/secrets/kubernetes.io/serviceaccount
total 0
lrwxrwxrwx 1 root root 13 Dec 15 01:17 ca.crt -> ..data/ca.crt
lrwxrwxrwx 1 root root 16 Dec 15 01:17 namespace -> ..data/namespace
lrwxrwxrwx 1 root root 12 Dec 15 01:17 token -> ..data/token
```

## 2. Opaque Secret

### 2.1. 创建secret

手动加密,基于base64加密

```shell
# 加密用户名和密码
[root@k8s-master volume]# echo -n "deemoprobe" | base64
ZGVlbW9wcm9iZQ==
[root@k8s-master volume]# echo -n "%TGB7ygvtest134242" | base64
JVRHQjd5Z3Z0ZXN0MTM0MjQy
[root@k8s-master volume]# mkdir secret;cd secret
# 创建配置文件
[root@k8s-master secret]# cat secret.yaml 
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: ZGVlbW9wcm9iZQ==
  password: JVRHQjd5Z3Z0ZXN0MTM0MjQy
# 创建并查看secret
[root@k8s-master secret]# kubectl apply -f secret.yaml 
secret/mysecret created
[root@k8s-master secret]# kubectl get secret
NAME                  TYPE                                  DATA   AGE
default-token-64lwm   kubernetes.io/service-account-token   3      34d
mysecret              Opaque                                2      2m46s
# 查看描述信息
[root@k8s-master secret]# kubectl describe secret mysecret
Name:         mysecret
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password:  18 bytes
username:  10 bytes
# 以YAML输出的方式查看详情
[root@k8s-master secret]# kubectl get secret mysecret -o yaml 
apiVersion: v1
data:
  password: JVRHQjd5Z3Z0ZXN0MTM0MjQy
  username: ZGVlbW9wcm9iZQ==
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"password":"JVRHQjd5Z3Z0ZXN0MTM0MjQy","username":"ZGVlbW9wcm9iZQ=="},"kind":"Secret","metadata":{"annotations":{},"name":"mysecret","namespace":"default"},"type":"Opaque"}
  creationTimestamp: "2020-12-15T05:33:40Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:password: {}
        f:username: {}
      f:metadata:
        f:annotations:
          .: {}
          f:kubectl.kubernetes.io/last-applied-configuration: {}
      f:type: {}
    manager: kubectl-client-side-apply
    operation: Update
    time: "2020-12-15T05:33:40Z"
  name: mysecret
  namespace: default
  resourceVersion: "359517"
  selfLink: /api/v1/namespaces/default/secrets/mysecret
  uid: 634f79b3-aeb2-4548-8e1a-54308bae1ce4
type: Opaque
```

### 2.2. 将Secret挂载到Volume中

```shell
# 创建yaml
[root@k8s-master secret]# vi pod_secret_volume.yaml
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
      secretName: mysecret
[root@k8s-master secret]# kubectl apply -f pod_secret_volume.yaml 
pod/pod-secret-volume created
[root@k8s-master secret]# kubectl get po pod-secret-volume
NAME                            READY   STATUS             RESTARTS   AGE
pod-secret-volume               1/1     Running            0          7s
# 进入secret Pod查看账户信息
[root@k8s-master secret]# kubectl exec -it pod-secret-volume -- /bin/sh
/ # ls /etc/secret/
password  username
/ # cat /etc/secret/username 
deemoprobe
/ # 
/ # cat /etc/secret/password 
%TGB7ygvtest134242
/ # 
```

由上可见,在pod中的secret信息实际已经被解密.

## 3. kubernetes.io/dockerconfigjson

首先需要有一个镜像仓库, 比如Docker Hub或者 Harbor搭建一个镜像仓库

通过以下方式去注册一个docker secret认证, 方可拉取私有镜像

kubectl create secret docker-registry myregistrysecret --docker-server='IP:Port' --docker-username='name' --docker-password='passwd'

```shell
# 比如镜像仓库地址为172.16.1.1:5000, 用户名admin, 密码Test123456, 私有镜像为myapp
[root@k8s-master secret]# kubectl create secret docker-registry myregistrysecret --docker-server='172.16.1.1:5000' --docker-username='admin' --docker-password='Test123456' 
secret/myregistrysecret created
[root@k8s-master secret]# kubectl get secret 
NAME                  TYPE                                  DATA   AGE
myregistrysecret      kubernetes.io/dockerconfigjson        1      8s
[root@k8s-master secret]# kubectl describe secret myregistrysecret
Name:         myregistrysecret
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  kubernetes.io/dockerconfigjson

Data
====
.dockerconfigjson:  112 bytes
# 创建pod, 拉取私有镜像
[root@k8s-master secret]# cat pod_secret_registry.yaml 
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
[root@k8s-master secret]# kubectl apply -f pod_secret_registry.yaml 
pod/pod-secret-registry created
[root@k8s-master secret]# kubectl describe pod pod-secret-registry
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
