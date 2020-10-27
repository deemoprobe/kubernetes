# kubernetes web 实例

## 实例说明

创建运行在Tomcat里面的Web APP, 实现JSP页面通过jdbc直接访问MySQL数据库在页面上展示数据.  
需要两个容器: Web APP 和 MySQL

## 创建MySQL

```shell
cd /etc/kubernetes/manifests
# 创建RC
vi mysql_rc.yaml

apiVersion: v1
# 定义为 RC (副本控制器)
# ReplicationSet目前在替代ReplicationController的写法,意义相同
kind: ReplicationController
metadata:
  # RC的名称,全局唯一
  name: mysql
spec:
  # 希望创建的pod个数
  replicas: 1
  selector:
    # 选择符合该标签的pod
    app: mysql
  # 根据模板下的定义来创建pod
  template:
    metadata:
      labels:
        # pod的标签,对应RC的selector
        app: mysql
    # 定义pod规则
    spec:
      # pod内容器的定义
      containers:
      # 容器名称
      - name: mysql
        # 容器所使用的的镜像(不指定版本的话就默认拉取最新版)
        # 由于最新版驱动的问题, 所以最好使用指定版本
        image: mysql:5.6
        ports:
        # 开放的端口号
        - containerPort: 3306
        # 容器环境变量
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "123456"

kubectl create -f mysql_rc.yaml

# 创建 SVC
vi mysql_svc.yaml

apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
    - port: 3306
  selector:
    app: mysql

kubectl create -f mysql_svc.yaml

[root@k8main manifests]# kubectl get pods,svc
NAME                         READY   STATUS    RESTARTS   AGE
pod/mysql-8d27z              1/1     Running   0          6m42s

NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
service/mysql        ClusterIP   192.168.68.128   <none>        3306/TCP       10s

# 查看pod状态
kubectl describe po mysql
```

## 创建 MyWeb APP

```shell
# 创建RC
vi myweb_rc.yaml

apiVersion: v1
kind: ReplicationController
metadata:
  name: myweb
spec:
  replicas: 2
  selector:
    app: myweb
  template:
    metadata:
      labels:
        app: myweb
    spec:
      containers:
      - name: myweb
        image: kubeguide/tomcat-app:v1
        ports:
        - containerPort: 8080
        env:
        - name: MYSQL_SERVICE_HOST
          # 这里的IP是名为MySQL的pod虚拟IP(CLUSTER-IP)
          value: 192.168.68.128
        - name: MYSQL_SERVICE_PORT
          value: "3306"

kubectl create -f myweb_rc.yaml

# 创建 SVC
vi myweb_svc.yaml

apiVersion: v1
kind: Service
metadata:
  name: myweb
spec:
  selector:
    app: myweb
  type: NodePort
  ports:
    # 本地服务的8080端口映射到外部端口30001
    - port: 8080
      nodePort: 30001

kubectl create -f myweb_svc.yaml
```

## 访问结果

![20201026143020](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20201026143020.png)

## 问题总结

1. MySQL版本需要选择5.6
2. 端口访问不通

```shell
# 先打开防火墙, 开放端口再关闭
systemctl start firewalld
firewall-cmd --zone=public --add-port=30001/tcp --permanent
firewall-cmd --reload
systemctl stop firewalld
```

## 台式机未配置

```shell
  cat /etc/sysctl.d/k8s.conf
  sysctl --system /etc/sysctl.d/k8s.conf
  sudo modprobe br_netfilter
  lsmod | grep br_netfilter

[root@k8s-master manifests]# kubectl get po,svc
NAME                         READY   STATUS    RESTARTS   AGE
pod/mysql-hdg66              1/1     Running   2          19h
pod/myweb-ctzhn              1/1     Running   2          19h
pod/myweb-dm94j              1/1     Running   2          19h
pod/nginx-6799fc88d8-xj4c4   1/1     Running   3          23h

NAME                 TYPE        CLUSTER-IP        EXTERNAL-IP   PORT(S)          AGE
service/kubernetes   ClusterIP   192.168.0.1       <none>        443/TCP          23h
service/mysql        ClusterIP   192.168.141.145   <none>        3306/TCP         19h
service/myweb        NodePort    192.168.115.169   <none>        8080:30001/TCP   19h
service/nginx        NodePort    192.168.94.34     <none>        80:30993/TCP     23h

[root@k8s-master manifests]# sysctl --system
* Applying /usr/lib/sysctl.d/00-system.conf ...
net.bridge.bridge-nf-call-ip6tables = 0
net.bridge.bridge-nf-call-iptables = 0
net.bridge.bridge-nf-call-arptables = 0
* Applying /usr/lib/sysctl.d/10-default-yama-scope.conf ...
kernel.yama.ptrace_scope = 0
* Applying /usr/lib/sysctl.d/50-default.conf ...
kernel.sysrq = 16
kernel.core_uses_pid = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.promote_secondaries = 1
net.ipv4.conf.all.promote_secondaries = 1
fs.protected_hardlinks = 1
fs.protected_symlinks = 1
* Applying /etc/sysctl.d/99-sysctl.conf ...
net.ipv4.ip_forward = 1
* Applying /etc/sysctl.d/k8s.conf ...
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
* Applying /etc/sysctl.conf ...
net.ipv4.ip_forward = 1

[root@k8s-master manifests]# netstat -lnt
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State
tcp        0      0 127.0.0.1:10248         0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.1:10249         0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.1:9099          0.0.0.0:*               LISTEN
tcp        0      0 192.168.43.236:2379     0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.1:2379          0.0.0.0:*               LISTEN
tcp        0      0 192.168.43.236:2380     0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.1:2381          0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.1:8080          0.0.0.0:*               LISTEN
tcp        0      0 0.0.0.0:30993           0.0.0.0:*               LISTEN
tcp        0      0 0.0.0.0:30001           0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.1:10257         0.0.0.0:*               LISTEN
tcp        0      0 0.0.0.0:179             0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.1:10259         0.0.0.0:*               LISTEN
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.1:25            0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.1:36636         0.0.0.0:*               LISTEN
tcp6       0      0 :::10250                :::*                    LISTEN
tcp6       0      0 :::6443                 :::*                    LISTEN
tcp6       0      0 :::10256                :::*                    LISTEN
tcp6       0      0 :::22                   :::*                    LISTEN
tcp6       0      0 ::1:25                  :::*                    LISTEN

[root@k8s-master manifests]# firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: ens32
  sources:
  services: dhcpv6-client ssh
  ports: 30001/tcp 8090/tcp
  protocols:
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:

# 两个文件里均配置net.ipv4.ip_forward=1
[root@k8s-master sysctl.d]# ls
99-sysctl.conf  k8s.conf

# node节点
[root@k8s-node1 ~]# netstat -lnt
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State
tcp        0      0 0.0.0.0:30993           0.0.0.0:*               LISTEN
tcp        0      0 0.0.0.0:30001           0.0.0.0:*               LISTEN
tcp        0      0 0.0.0.0:179             0.0.0.0:*               LISTEN
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.1:25            0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.1:10248         0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.1:10249         0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.1:38858         0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.1:9099          0.0.0.0:*               LISTEN
tcp6       0      0 :::10256                :::*                    LISTEN
tcp6       0      0 :::22                   :::*                    LISTEN
tcp6       0      0 ::1:25                  :::*                    LISTEN
tcp6       0      0 :::10250                :::*                    LISTEN

# node节点仅配置了下面这个流量链路
[root@k8s-node1 sysctl.d]# cat k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1

# 发现99---conf是sysctl.conf的软连接
lrwxrwxrwx. 1 root root 14 Oct 21 01:34 /etc/sysctl.d/99-sysctl.conf -> ../sysctl.conf
```
