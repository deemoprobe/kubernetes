# Kubernetes资源配置之namespace

更改当前默认的namesapcce: `kubectl config set-context --current --namespace=<命名空间名称>`

```shell
# 更改默认namesapce为kube-system
[root@k8s-master /]# kubectl config set-context --current --namespace=kube-system
Context "kubernetes-admin@kubernetes" modified.
[root@k8s-master /]# kubectl get po
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-5c6f6b67db-4gwqx   1/1     Running   23         43d
calico-node-g5r6h                          1/1     Running   22         43d
calico-node-tw24f                          1/1     Running   23         43d
coredns-6d56c8448f-m92h8                   1/1     Running   23         43d
coredns-6d56c8448f-wh66t                   1/1     Running   23         43d
etcd-k8s-master                            1/1     Running   23         43d
kube-apiserver-k8s-master                  1/1     Running   29         43d
kube-controller-manager-k8s-master         1/1     Running   25         43d
kube-proxy-2ld6t                           1/1     Running   23         43d
kube-proxy-bx9jg                           1/1     Running   22         43d
kube-scheduler-k8s-master                  1/1     Running   25         43d
# 更改默认namesapce为default
[root@k8s-master /]# kubectl config set-context --current --namespace=default
Context "kubernetes-admin@kubernetes" modified.
[root@k8s-master /]# kubectl get po
NAME                      READY   STATUS             RESTARTS   AGE
busybox-k8s-master        1/1     Running            37         47h
look-svc-env-k8s-master   0/1     CrashLoopBackOff   318        47h
nginx-k8s-master          1/1     Running            8          47h
pod-dns-k8s-master        1/1     Running            7          47h
```

安装namespace切换工具

```shell
curl -L https://github.com/ahmetb/kubectx/releases/download/v0.9.1/kubens -o /bin/kubens
chmod +x /bin/kubens
kubens <命名空间名称>
[root@k8s-master /]# kubens -h
USAGE:
  kubens                    : list the namespaces in the current context
  kubens <NAME>             : change the active namespace of current context
  kubens -                  : switch to the previous namespace in this context
  kubens -c, --current      : show the current namespace
  kubens -h,--help          : show this message
# 列出所有namespace
[root@k8s-master /]# kubens
default
kube-node-lease
kube-public
kube-system
test
# 查看当前namespace
[root@k8s-master /]# kubens -c
test
# 切换为default
[root@k8s-master /]# kubens default
Context "kubernetes-admin@kubernetes" modified.
Active namespace is "default".
# 切换为上一个namespace
[root@k8s-master /]# kubens -
Context "kubernetes-admin@kubernetes" modified.
Active namespace is "test".
# 查看当前namespace
[root@k8s-master /]# kubens -c
test
```
