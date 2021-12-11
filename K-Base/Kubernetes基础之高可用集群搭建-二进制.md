# Kubernetes基础之高可用集群搭建-kubeadm方式

## 环境说明

- 宿主机系统: Windows 10
- 虚拟机版本: VMware® Workstation 16 Pro
- IOS镜像版本: CentOS Linux release 7.9.2009
- 集群操作用户: root

CentOS7虚拟机安装和配置静态IP请参考博客文章：[VMWARE WORKSTATION安装CENTOS7并配置静态IP](http://www.deemoprobe.com/principle/vmware-workstation%e5%ae%89%e8%a3%85centos7%e5%b9%b6%e9%85%8d%e7%bd%ae%e9%9d%99%e6%80%81ip/)

## 资源分配

### 网段划分

Kubernetes集群需要规划三个网段：

- 宿主机网段：Kubernetes集群节点的网段
- Pod网段：集群内Pod的网段，相当于容器的IP
- Service网段：集群内服务发现使用的网段，service用于集群容器通信

生产环境根据申请到的IP资源进行分配即可，原则是三个网段不要有交叉。本文虚拟机练习环境IP地址段分配如下：

- 宿主机网段：`192.168.43.18*/24`
- Pod网段：`172.16.0.1/12`
- Service：`10.96.0.0/12`

### 节点分配

本次实验采用3管理节点2工作节点的高可用Kubernetes集群模式:

- k8s-master01/k8s-master02/k8s-master03 集群的Master节点
- 三个master节点同时做etcd集群
- k8s-node01/k8s-node02 集群的Node节点
- k8s-master-vip: 192.168.43.182,是做高可用k8s-master01~3的虚拟IP,不占用物理资源

|  主机节点名称  |       IP       | CPU核心数 | 内存大小 | 磁盘大小 |
| :------------: | :------------: | :-------: | :------: | :------: |
| k8s-master-vip | 192.168.43.182 |     /     |    /     |    /     |
|  k8s-master01  | 192.168.43.183 |     2     |    2G    |   40G    |
|  k8s-master02  | 192.168.43.184 |     2     |    2G    |   40G    |
|  k8s-master03  | 192.168.43.185 |     2     |    2G    |   40G    |
|   k8s-node01   | 192.168.43.186 |     2     |    2G    |   40G    |
|   k8s-node02   | 192.168.43.187 |     2     |    2G    |   40G    |

## 操作步骤

小括号注释说明：

- ALL 所有节点都要执行
- Master 只需要在master节点(k8s-master01/k8s-master02/k8s-master03)执行
- Node 只需要在node节点(k8s-node01/k8s-node02)执行

### 添加主机信息(ALL)

```bash
# 以k8s-master01为例
[root@k8s-master01 ~]# cat << EOF >> /etc/hosts
192.168.43.182    k8s-master-vip
192.168.43.183    k8s-master01
192.168.43.184    k8s-master02
192.168.43.185    k8s-master03
192.168.43.186    k8s-node01
192.168.43.187    k8s-node02
EOF
```

### 关闭防火墙/swap/SELinux/重置iptables(ALL)

```bash
# 直接执行下面命令
systemctl stop firewalld
systemctl disable firewalld
systemctl disable --now dnsmasq
systemctl disable --now NetworkManager

swapoff -a
sed -i '/swap/s/^\(.*\)$/#\1/g' /etc/fstab

setenforce 0
sed -i "s/=enforcing/=disabled/g" /etc/sysconfig/selinux
```

### 更新源并升级内核(ALL)

```bash
# 默认的yum源太慢，更新为阿里源，同时用sed命令删除包含下面不需要的两个URL的行
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
sed -i -e '/mirrors.cloud.aliyuncs.com/d' -e '/mirrors.aliyuncs.com/d' /etc/yum.repos.d/CentOS-Base.repo

# 安装常用工具包（在需要时安装也可以，通常一起装了比较省事）
yum install wget jq psmisc vim net-tools telnet yum-utils device-mapper-persistent-data lvk8s-master02 git -y

# 升级内核，4.17以下的内核cgroup存在内存泄漏的BUG，具体分析过程浏览器搜一下“Kubernetes集群为什么要升级内核”会有一大波文章
cd /root
wget http://193.49.22.109/elrepo/kernel/el7/x86_64/RPMS/kernel-ml-devel-4.19.12-1.el7.elrepo.x86_64.rpm
wget http://193.49.22.109/elrepo/kernel/el7/x86_64/RPMS/kernel-ml-4.19.12-1.el7.elrepo.x86_64.rpm
# 所有节点安装内核
cd /root && yum localinstall -y kernel-ml*
# 所有节点更改内核启动顺序
grub2-set-default  0 && grub2-mkconfig -o /etc/grub2.cfg
grubby --args="user_namespace.enable=1" --update-kernel="$(grubby --default-kernel)"
# 检查默认内核是不是4.19，并重启节点
grubby --default-kernel
reboot

# 检查内核是不是4.19
uname -a
# （可选）删除老版本的内核，避免以后被升级取代默认的开机4.19内核
rpm -qa | grep kernel
yum remove -y kernel-3*

# 升级系统软件包（如果跳过内核升级加参数 --exclude=kernel*）
yum update -y

# 安装IPVS内核模块，由于IPVS在资源消耗和性能上均已明显优于iptables，所以推荐开启
# 具体原因可参考官网介绍 https://kubernetes.io/zh/blog/2018/07/09/ipvs-based-in-cluster-load-balancing-deep-dive/
yum install ipvsadm ipset sysstat conntrack libseccomp -y
# 加载模块，最后一条4.18以下内核使用nf_conntrack_ipv4，4.19已改为nf_conntrack
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack
# 编写参数文件
cat << EOF > /etc/modules-load.d/ipvs.conf 
ip_vs
ip_vs_lc
ip_vs_wlc
ip_vs_rr
ip_vs_wrr
ip_vs_lblc
ip_vs_lblcr
ip_vs_dh
ip_vs_sh
ip_vs_fo
ip_vs_nq
ip_vs_sed
ip_vs_ftp
ip_vs_sh
nf_conntrack
ip_tables
ip_set
xt_set
ipt_set
ipt_rpfilter
ipt_REJECT
ipip
EOF
# 加载
systemctl enable --now systemd-modules-load.service
# 自定义内核参数配置文件
cat << EOF > /etc/sysctl.d/kubernetes.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
fs.may_detach_mounts = 1
net.ipv4.conf.all.route_localnet = 1
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
fs.file-max=52706963
fs.nr_open=52706963
net.netfilter.nf_conntrack_max=2310720
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl =15
net.ipv4.tcp_max_tw_buckets = 36000
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_orphans = 327680
net.ipv4.tcp_orphan_retries = 3
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.ip_conntrack_max = 65536
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.tcp_timestamps = 0
net.core.somaxconn = 16384
EOF
# 加载
sysctl --system
# 重启查看PIVS模块是否依旧加载
reboot
lsmod | grep ip_vs

# 配置ntpdate,同步服务器时间
rpm -ivh http://mirrors.wlnmp.com/centos/wlnmp-release-centos.noarch.rpm
yum install ntpdate -y
# 同步时区和时间
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
echo 'Asia/Shanghai' >/etc/timezone
ntpdate time2.aliyun.com

# 配置limits
cat << EOF >> /etc/security/limits.conf
* soft nofile 655360
* hard nofile 131072
* soft nproc 655350
* hard nproc 655350
* soft memlock unlimited
* hard memlock unlimited
EOF

# 配置免密登录, k8s-master01到其他节点
# 先生成认证文件
ssh-keygen -t rsa
# 拷贝公钥信息到其他节点，同时认证一次各个节点的root密码，以后就可以免密ssh到其他节点
for i in k8s-master02 k8s-master03 k8s-node01 k8s-node02;do ssh-copy-id -i .ssh/id_rsa.pub $i;done
```

### 3.6. 部署Docker(ALL)

```bash
# 卸载已存在docker，新机器的话这步可以忽略
yum remove -y docker \
              docker-client \
              docker-client-latest \
              docker-common \
              docker-latest \
              docker-latest-logrotate \
              docker-logrotate \
              docker-engine

yum remove -y docker-ce docker-ce-cli containerd.io

# 设置docker仓库
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 安装最新版本docker
yum install docker-ce docker-ce-cli containerd.io -y

# （可选）如果想要安装指定版本docker，先查询一下。安装指定版本，例如20.10.9-3.el7
yum list docker-ce docker-ce-cli --showduplicates | grep "^doc" | sort -r
yum install docker-ce-20.10.9-3.el7 docker-ce-cli-20.10.9-3.el7 containerd.io -y

# 加入开机启动并启动
systemctl enable docker
systemctl start docker

# 测试运行hello-world镜像并查看docker版本信息
docker run hello-world
docker version

# 配置阿里docker镜像加速器，阿里云(登录账号-->点击管理控制台-->搜索容器镜像服务-->镜像工具-->镜像加速器-->复制加速器地址)
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

```bash
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

```bash
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

```bash
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

```bash
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

```bash
# 根据kubeadm init最后提示token
kubeadm join 192.168.3.13:6443 --token vmbhd0.e4cszyn6ozqv9tet \
  --discovery-token-ca-cert-hash sha256:c8fd072fd90c9ddbae3b3945ba0120abf5ed6777de3354a6eb4814a93ed1b844
```

## 4. 测试集群

master节点执行

```bash
# 查看集群状态
kubectl get nodes
NAME         STATUS   ROLES    AGE     VERSION
k8s-master   Ready    master   52m     v1.19.3
k8s-node1    Ready    <none>   2k8s-master029s   v1.19.3
k8s-node2    Ready    <none>   2k8s-master03s    v1.19.3

# 增加node节点的节点role名称
kubectl label nodes k8s-node1 node-role.kubernetes.io/node=
# 删除node节点的节点role名称
kubectl label nodes k8s-node1 node-role.kubernetes.io/node-

# 添加标签后查看集群状态
kubectl get nodes
NAME         STATUS   ROLES    AGE     VERSION
k8s-master   Ready    master   54m     v1.19.3
k8s-node1    Ready     node    4m57s   v1.19.3
k8s-node2    Ready     node    4k8s-master031s    v1.19.3

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
pod/nginx-f89759699-9265g   1/1     Running   0          5k8s-master030s

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
service/kubernetes   ClusterIP    x.x.x.x         <none>      443/TCP        12m
service/nginx        NodePort     x.x.x.x         <none>      80:30086/TCP   1k8s-master015s

```

- 在浏览器输入<http://IP:30086/> 访问nginx

![20201027135155](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20201027135155.png)
