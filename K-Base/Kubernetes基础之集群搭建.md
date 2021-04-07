# Kubernetes基础之高可用集群搭建-kubeadm方式

- `kubeadm` 部署Kubernetes集群工具
- `kubectl` Kubernetes集群管理工具(Master节点上安装即可)
- `kubelet` Kubernetes集群中作为后台进程存在,管理pod和容器生命周期

## 1. 环境说明

- 虚拟化平台: VirtualBox 6.1.18
- 操作系统: CentOS Linux release 7.9.2009
- 操作用户: root
- 电脑型号: Lenovo Yoga 14s 2021
- CPU: AMD Ryzen 7 4800H (8c16t)
- RAM: 16G

### 1.1. 安装虚拟机

安装VirtualBox虚拟机,去[Oracle VirtualBox官网](https://www.virtualbox.org/wiki/Downloads)下载介质包,一路下一步安装即可.

### 1.2. 安装CentOS

准备好CentOS7的镜像文件,去[清华大学开源软件镜像站](https://mirror.tuna.tsinghua.edu.cn/)搜索"centos"下载想要的版本即可.

VirtualBox虚拟机安装CentOS系统网上一大堆,自行搜索安装即可.如果对Linux比较熟悉,安装时选择"Minimal Install"即可,后面需要什么工具针对性安装即可,桌面版操作简单(适合新手)但很多功能用不到,实在没必要把资源浪费在不必要的地方.

### 1.3. 配置静态IP

由于Kubernetes集群对网络环境的稳定性要求还是比较高的,所以推荐配一下CentOS虚拟机静态IP.

#### 1.3.1. 确定自己网络IP段

Win+R输入cmd进入Windows命令行,查看自己网络的IP段,据此分配静态IP

```shell
# 比如我连的WiFi,查看下面三个参数即可
# CMD中执行ipconfig
C:\Users\deemoprobe>ipconfig
...
无线局域网适配器 WLAN:

   连接特定的 DNS 后缀 . . . . . . . :
   ...
   IPv4 地址 . . . . . . . . . . . . : 192.168.43.219
   子网掩码  . . . . . . . . . . . . : 255.255.255.0
   默认网关. . . . . . . . . . . . . : 192.168.43.1
...
```

#### 1.3.2. VirtualBox设置

选中对应的虚拟机,设置->网络,启用网卡1和网卡2

网卡1使用默认的"网络地址转换(NAT)"模式

![20210406163342](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20210406163342.png)

网卡2使用"桥接网卡"模式,选择你在使用的网卡

![20210406163436](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20210406163436.png)

#### 1.3.3. 静态IP配置

进入虚拟机,编辑网络配置文件,主要注意网关和IP段和之前查的Windows本地下相同网段即可

```shell
# 以m1为例,找到ifcfg-en开头的网络配置文件即可
[root@m1 network-scripts]# pwd
/etc/sysconfig/network-scripts
[root@m1 network-scripts]# ls
ifcfg-enp0s3  ifdown-bnep  ifdown-isdn    ifdown-sit       ifup          ifup-ippp  ifup-plusb   ifup-sit       ifup-wireless
ifcfg-enp0s8  ifdown-eth   ifdown-post    ifdown-Team      ifup-aliases  ifup-ipv6  ifup-post    ifup-Team      init.ipv6-global
ifcfg-lo      ifdown-ippp  ifdown-ppp     ifdown-TeamPort  ifup-bnep     ifup-isdn  ifup-ppp     ifup-TeamPort  network-functions
ifdown        ifdown-ipv6  ifdown-routes  ifdown-tunnel    ifup-eth      ifup-plip  ifup-routes  ifup-tunnel    network-functions-ipv6
# 网卡1配置文件ifcfg-enp0s3 不需要改什么,查看一下即可
[root@m1 network-scripts]# cat ifcfg-enp0s3 
TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="dhcp"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="enp0s3"
UUID="d7d6337d-ee3d-4313-b596-447cf67011f8"
DEVICE="enp0s3"
ONBOOT="yes"
# 主要修改网卡2的配置文件 我这里是ifcfg-enp0s8,参考我的配置文件
[root@m1 network-scripts]# cat ifcfg-enp0s8
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=enp0s8
UUID=fa11e783-2f96-489a-b904-98166afbd7fb
DEVICE=enp0s8
ONBOOT=yes
IPADDR=192.168.43.21
PREFIX=24
NETMASK=255.255.255.0
GATEWAY=192.168.43.1
IPV6_PRIVACY=no

# 修改后重启网络服务
[root@m1 network-scripts]# systemctl restart network

# 如果重启后没有生效,可以尝试给网卡2的网络配置文件重新申请一个UUID
# 生成后将下面的UUID配置到ifcfg-enp0s8,重启网络服务
[root@m1 network-scripts]# uuidgen
af9d7267-df0f-468d-a7a5-657efd623829
```

测试一下:

```shell
# 1.虚拟机访问外网
[root@m1 network-scripts]# ping baidu.com
PING baidu.com (39.156.69.79) 56(84) bytes of data.
64 bytes from 39.156.69.79 (39.156.69.79): icmp_seq=1 ttl=48 time=49.0 ms
64 bytes from 39.156.69.79 (39.156.69.79): icmp_seq=2 ttl=48 time=88.8 ms
64 bytes from 39.156.69.79 (39.156.69.79): icmp_seq=3 ttl=48 time=45.7 ms
64 bytes from 39.156.69.79 (39.156.69.79): icmp_seq=4 ttl=48 time=49.3 ms
^C
--- baidu.com ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3007ms
rtt min/avg/max/mdev = 45.739/58.252/88.831/17.712 ms

# 2.本地访问虚拟机
C:\Users\deemoprobe>ping 192.168.43.21

正在 Ping 192.168.43.21 具有 32 字节的数据:
来自 192.168.43.21 的回复: 字节=32 时间<1ms TTL=64
来自 192.168.43.21 的回复: 字节=32 时间<1ms TTL=64
来自 192.168.43.21 的回复: 字节=32 时间<1ms TTL=64

192.168.43.21 的 Ping 统计信息:
    数据包: 已发送 = 3，已接收 = 3，丢失 = 0 (0% 丢失)，
往返行程的估计时间(以毫秒为单位):
    最短 = 0ms，最长 = 0ms，平均 = 0ms
```

注:本文静态IP配置参考文章 <https://blog.csdn.net/qq_38669394/article/details/80051356>

## 2. 资源分配

### 2.1. 节点准备

本次实验采用3主2从的高可用Kubernetes集群模式:

- m1/m2/m3 集群的Master节点
- s1/s2 集群的Node节点

| 主机节点名称 |      IP       | CPU核心数 | 内存大小 | 磁盘大小 |
| :----------: | :-----------: | :-------: | :------: | :------: |
|      m1      | 192.168.43.21 |     2     |    3G    |   40G    |
|      m2      | 192.168.43.22 |     2     |    3G    |   40G    |
|      m3      | 192.168.43.23 |     2     |    3G    |   40G    |
|      s1      | 192.168.43.24 |     2     |    3G    |   40G    |
|      s2      | 192.168.43.25 |     2     |    3G    |   40G    |

## 3. 操作步骤

- ALL 所有节点都要执行
- Master 只需要在master节点(m1/m2/m3)执行
- Node 只需要在node节点(s1/s2)执行

如果**最后三步**安装过程中有报错, 先根据报错信息尝试解决, 无法解决时, 可以`kubeadm reset --force`重置集群重新配置

### 3.4. 添加主机信息(ALL)

```shell
# 以m1为例
[root@m1 ~]# vi /etc/hosts
...
192.168.43.21    m1
192.168.43.22    m2
192.168.43.23    m3
192.168.43.24    s1
192.168.43.25    s2
```

### 3.1. 关闭防火墙/swap/SELinux/重置iptables(ALL)

```shell
# 直接执行下面命令
systemctl stop firewalld
systemctl disable firewalld

swapoff -a
sed -i '/swap/s/^\(.*\)$/#\1/g' /etc/fstab

setenforce 0
sed -i "s/=enforcing/=disabled/g" /etc/sysconfig/selinux

iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat && iptables -P FORWARD ACCEPT
```

### 3.5. 系统参数配置(ALL)

```shell
# 制作配置文件
$ cat > /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
vm.swappiness=0
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
EOF
# 生效文件
$ sysctl -p /etc/sysctl.d/kubernetes.conf
```

### 3.6. 部署Docker(ALL)

```shell
# 卸载已存在docker
yum remove docker \
            docker-client \
            docker-client-latest \
            docker-common \
            docker-latest \
            docker-latest-logrotate \
            docker-logrotate \
            docker-engine

# 设置docker仓库
yum install -y yum-utils
yum-config-manager \
  --add-repo \
  https://download.docker.com/linux/centos/docker-ce.repo

# 安装docker-ce
yum install docker-ce docker-ce-cli containerd.io

# 加入开机启动并启动
systemctl enable docker
systemctl start docker

# 测试运行并查看版本信息
docker run hello-world
docker version

# 配置阿里docker镜像加速器
# ["https://ynirk4k5.mirror.aliyuncs.com"]中"ynirk4k5"根据自己申请的阿里镜像加速器id来配置
# docker文件驱动改成 systemd
cat <<EOF > /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirror": ["https://ynirk4k5.mirror.aliyuncs.com"]
}
EOF

# 重启docker
systemctl restart docker
# 如果启动失败,强制加载再启动试试
systemctl reset-failed docker
systemctl restart docker

# 查看docker配置信息
docker info
docker info | grep Driver
```

### 3.7. 配置kubernetes镜像源(ALL)

```shell
# 此处选择的是阿里云镜像源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
       http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

### 3.8. 安装kubernetes(ALL)

一般kebectl在master节点安装即可,node节点装不装均可

```shell
# 查看可以安装的版本号
yum list kubeadm --showduplicates | sort -r
# 不指定版本的话默认安装最新版本安装
yum install -y kubelet kubeadm kubectl
# 如果gpg检查失败,可以跳过gpg检查进行安装
yum install -y --nogpgcheck kubelet kubeadm kubectl
# 开机启动
systemctl enable kubelet
# 先不启动kubelet,因为会启动失败,提示初始化文件不存在,需要先执行kubeadm init完成初始化
```

### 3.9. 部署k8s-master(Master)

在master节点上执行

```shell
# 初始化
kubeadm init --kubernetes-version=1.19.3  \
--apiserver-advertise-address=192.168.3.13   \
--image-repository registry.aliyuncs.com/google_containers  \
--service-cidr=192.168.0.0/16 \
--pod-network-cidr=192.168.0.0/16
# 记录好输出信息Your Kubernetes control-plane has initialized successfully!后十几行

# 按照提示执行下面步骤
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# 查看docker镜像,可以看到kube..和etcd等镜像
docker images

# kube-apiserver默认只启动安全访问接口6443，而不启动非安装访问接口8080，kubectl是通过8080端口访问k8s kubelet的，所以要修改配置文件，使其支持8080端口访问
vim /etc/kubernetes/manifests/kube-apiserver.yaml
- --insecure-port=8080

# 启动kubelet
systemctl start kubelet

# 查看集群状态信息
kubectl get nodes
NAME         STATUS     ROLES    AGE   VERSION
k8s-master   NotReady   master   28m   v1.19.3
```

### 3.10. 配置calico网络(Master)

在master节点上执行

网络方案也可以选择其他(例如：flannel等)

```shell
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
# 查看calico网络状态, STATUS为Running后查看集群信息
kubectl get pods -n kube-system
kubectl get nodes
NAME         STATUS   ROLES    AGE   VERSION
k8s-master   Ready    master   45m   v1.19.3

# 查看配置各组件信息configmap
[root@k8s-master ~]# kubectl get -n kube-system configmap
NAME                                 DATA   AGE
calico-config                        4      29m
coredns                              1      29m
extension-apiserver-authentication   6      29m
kube-proxy                           2      29m
kubeadm-config                       2      29m
kubelet-config-1.19                  1      29m
```

### 3.11. node节点加入集群(Node)

在node节点上执行

```shell
# 根据kubeadm init最后提示token
kubeadm join 192.168.3.13:6443 --token vmbhd0.e4cszyn6ozqv9tet \
  --discovery-token-ca-cert-hash sha256:c8fd072fd90c9ddbae3b3945ba0120abf5ed6777de3354a6eb4814a93ed1b844
```

## 4. 测试集群

master节点执行

```shell
# 查看集群状态
kubectl get nodes
NAME         STATUS   ROLES    AGE     VERSION
k8s-master   Ready    master   52m     v1.19.3
k8s-node1    Ready    <none>   2m29s   v1.19.3
k8s-node2    Ready    <none>   2m3s    v1.19.3

# 增加node节点的节点role名称
kubectl label nodes k8s-node1 node-role.kubernetes.io/node=
# 删除node节点的节点role名称
kubectl label nodes k8s-node1 node-role.kubernetes.io/node-

# 添加标签后查看集群状态
kubectl get nodes
NAME         STATUS   ROLES    AGE     VERSION
k8s-master   Ready    master   54m     v1.19.3
k8s-node1    Ready     node    4m57s   v1.19.3
k8s-node2    Ready     node    4m31s    v1.19.3

# 测试namespace
kubectl get namespace
kubectl create namespace test
kubectl get namespace
kubectl delete namespace test

# 创建nginx实例并开放端口
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get pod,svc
NAME                        READY   STATUS    RESTARTS   AGE
pod/nginx-f89759699-9265g   1/1     Running   0          5m30s

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
service/kubernetes   ClusterIP    x.x.x.x         <none>      443/TCP        12m
service/nginx        NodePort     x.x.x.x         <none>      80:30086/TCP   1m15s

```

- 在浏览器输入<http://IP:30086/> 访问nginx

![20201027135155](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20201027135155.png)
