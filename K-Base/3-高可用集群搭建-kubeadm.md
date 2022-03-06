# Kubernetes基础之高可用集群搭建-kubeadm方式

## 环境说明

- 宿主机系统：Windows 10
- 虚拟机版本：VMware® Workstation 16 Pro
- IOS镜像版本：CentOS Linux release 7.9.2009
- 集群操作用户：root
- Kubernetes版本：1.23.0
- Runtime：Docker

CentOS7安装请参考博客文章：[LINUX之VMWARE WORKSTATION安装CENTOS-7](http://www.deemoprobe.com/standard/vmware-centos7/)

## 资源分配

### 网段划分

Kubernetes集群需要规划三个网段：

- 宿主机网段：Kubernetes集群节点的网段
- Pod网段：集群内Pod的网段，相当于容器的IP
- Service网段：集群内服务发现使用的网段，service用于集群容器通信

生产环境根据申请到的IP资源进行分配即可，原则是三个网段不允许有重合IP。IP网段计算可以参考：[在线IP地址计算](http://tools.jb51.net/aideddesign/ip_net_calc/)。本文虚拟机练习环境IP地址网段分配如下：

- 宿主机网段：`192.168.43.1/24`
- Pod网段：`172.16.0.0/12`
- Service：`10.96.0.0/12`

### 节点分配

采用`3管理节点2工作节点`的高可用Kubernetes集群模式：

- k8s-master01/k8s-master02/k8s-master03 集群的Master节点
- 三个master节点同时做etcd集群
- k8s-node01/k8s-node02 集群的Node节点
- k8s-master-vip做高可用k8s-master01~03的VIP，不占用物理资源

| 主机节点名称   |       IP       | CPU核心数 | 内存大小 | 磁盘大小 |
| :------------- | :------------: | :-------: | :------: | :------: |
| k8s-master-vip | 192.168.43.182 |     /     |    /     |    /     |
| k8s-master01   | 192.168.43.183 |     2     |    2G    |   40G    |
| k8s-master02   | 192.168.43.184 |     2     |    2G    |   40G    |
| k8s-master03   | 192.168.43.185 |     2     |    2G    |   40G    |
| k8s-node01     | 192.168.43.186 |     2     |    2G    |   40G    |
| k8s-node02     | 192.168.43.187 |     2     |    2G    |   40G    |

## 操作步骤

标题后小括号注释表明操作范围：

- ALL 所有节点（k8s-master01/k8s-master02/k8s-master03/k8s-node01/k9s-node02）执行
- Master 只需要在master节点（k8s-master01/k8s-master02/k8s-master03）执行
- Node 只需要在node节点（k8s-node01/k8s-node02）执行
- 已标注的个别命令只需要在某一台机器执行，会在操作前说明
- 未标注的会在操作时说明

使用`cat << EOF >> file`添加文件内容注意里面如果有变量定义，需要将变量转义，例如`$Parameter`应改写为`\$Parameter`；同时注意`>`、`>>`。

### 准备工作(ALL)

添加主机信息、关闭防火墙、关闭swap、关闭SELinux、dnsmasq、NetworkManager

```bash
# 添加主机信息
cat << EOF >> /etc/hosts
192.168.43.182    k8s-master-vip
192.168.43.183    k8s-master01
192.168.43.184    k8s-master02
192.168.43.185    k8s-master03
192.168.43.186    k8s-node01
192.168.43.187    k8s-node02
EOF
# 关闭防火墙、dnsmasq、NetworkManager，--now参数表示关闭服务并移除开机自启
# 这些服务是否可以关闭视情况而定，本文是虚拟机实践，没有用到这些服务
systemctl disable --now firewalld
systemctl disable --now dnsmasq
systemctl disable --now NetworkManager
# 关闭swap，并注释fstab文件swap所在行
swapoff -a
sed -i '/swap/s/^\(.*\)$/#\1/g' /etc/fstab
# 关闭SELinux，并更改selinux配置文件
setenforce 0
sed -i "s/=enforcing/=disabled/g" /etc/selinux/config
```

值得注意的是`/etc/sysconfig/selinux`文件是`/etc/selinux/config`文件的软连接，用`sed -i`命令修改软连接文件会破坏软连接属性，将`/etc/sysconfig/selinux`变为一个独立的文件，即使该文件被修改了，但源文件`/etc/selinux/config`配置是没变的。此外，使用vim等编辑器编辑源文件或链接文件（编辑模式不会修改文件属性）修改也可以。软链接原理可参考博客：[LINUX之INODE详解](http://www.deemoprobe.com/yunv/inode/)

### 必要操作(ALL)

```bash
# 默认的yum源太慢，更新为阿里源，同时用sed命令删除文件中不需要的两个URL的行
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
sed -i -e '/mirrors.cloud.aliyuncs.com/d' -e '/mirrors.aliyuncs.com/d' /etc/yum.repos.d/CentOS-Base.repo

# 安装常用工具包
yum install wget jq psmisc vim net-tools telnet yum-utils device-mapper-persistent-data lvm2 git -y

# 配置阿里Kubernetes源
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
# 配置阿里docker源
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 配置ntpdate，同步服务器时间
rpm -ivh http://mirrors.wlnmp.com/centos/wlnmp-release-centos.noarch.rpm
yum install ntpdate -y
# 同步时区和时间
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
echo 'Asia/Shanghai' >/etc/timezone
ntpdate time2.aliyun.com
# 可以加入计划任务，保证集群时钟是一致的
# /var/spool/cron/root文件也是crontab -e写入的文件
# crontab执行日志查看可用：tail -f /var/log/cron
cat << EOF >> /var/spool/cron/root
*/5 * * * * /usr/sbin/ntpdate time2.aliyun.com
EOF

# 保证文件句柄不会限制集群的可持续发展，配置limits
ulimit -SHn 65500
cat << EOF >> /etc/security/limits.conf
* soft nofile 65500
* hard nofile 65500
* soft nproc 65500
* hard nproc 65500
* soft memlock unlimited
* hard memlock unlimited
EOF

# 配置免密登录，k8s-master01到其他节点
# 生成密钥对（在k8s-master01节点配置即可）
ssh-keygen -t rsa
# 拷贝公钥到其他节点，首次需要认证一下各个节点的root密码，以后就可以免密ssh到其他节点
for i in k8s-master02 k8s-master03 k8s-node01 k8s-node02;do ssh-copy-id -i .ssh/id_rsa.pub $i;done

# 克隆二进制仓库文件（k8s-master01上操作即可）
cd root;git clone https://gitee.com/deemoprobe/k8s-ha-install.git

cd k8s-ha-install;git branch -a
* master
  remotes/origin/HEAD -> origin/master
  remotes/origin/manual-installation
  remotes/origin/manual-installation-v1.16.x
  remotes/origin/manual-installation-v1.17.x
  remotes/origin/manual-installation-v1.18.x
  remotes/origin/manual-installation-v1.19.x
  remotes/origin/manual-installation-v1.20.x
  remotes/origin/manual-installation-v1.20.x-csi-hostpath
  remotes/origin/manual-installation-v1.21.x
  remotes/origin/manual-installation-v1.22.x
  remotes/origin/manual-installation-v1.23.x
  remotes/origin/master
# 可以切换到需要版本的分支中获取配置文件
git checkout manual-installation-v1.22.x

# 所有节点系统升级
yum update --exclude=kernel* -y
```

升级内核，4.17以下的内核cgroup存在内存泄漏的BUG，具体分析过程浏览器搜`Kubernetes集群为什么要升级内核`会有很多文章讲解

内核备用下载（建议下载到本地后上传到服务器，尽量不用`wget`）：

- [kernel-ml-devel-4.19.12-1.el7.elrepo.x86_64.rpm](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/repo/kernel-ml-devel-4.19.12-1.el7.elrepo.x86_64.rpm?versionId=CAEQNBiBgMDJiNbj.hciIDUyMDZlYjU5YzIwMzQ0MmNhNzBmNjBiMDY3Yjc0Y2Jl)
- [kernel-ml-4.19.12-1.el7.elrepo.x86_64.rpm](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/repo/kernel-ml-4.19.12-1.el7.elrepo.x86_64.rpm?versionId=CAEQNBiBgMDNiNbj.hciIGQ0M2RmZDAwNDhlNjQyNjE5MTE4MDk1OGU4OThiNWY4)

```bash
# 下载4.19版本内核，如果无法下载，可以用上面提供的备用下载
cd /root
wget http://193.49.22.109/elrepo/kernel/el7/x86_64/RPMS/kernel-ml-devel-4.19.12-1.el7.elrepo.x86_64.rpm
wget http://193.49.22.109/elrepo/kernel/el7/x86_64/RPMS/kernel-ml-4.19.12-1.el7.elrepo.x86_64.rpm

# 可以在k8s-master01节点下载后，免密传到其他节点
for i in k8s-master02 k8s-master03 k8s-node01 k8s-node02;do scp kernel-ml-* $i:/root;done

# 所有节点安装内核
cd /root && yum localinstall -y kernel-ml*
# 所有节点更改内核启动顺序
grub2-set-default  0 && grub2-mkconfig -o /etc/grub2.cfg
grubby --args="user_namespace.enable=1" --update-kernel="$(grubby --default-kernel)"
# 查看默认内核，并重启节点
grubby --default-kernel
reboot

# 确认内核版本
uname -a
# （可选）删除老版本的内核，避免以后被升级取代默认的开机4.19内核
rpm -qa | grep kernel
yum remove -y kernel-3*

# 升级系统软件包（如果跳过内核升级加参数 --exclude=kernel*）
yum update -y

# 安装IPVS相关工具，由于IPVS在资源消耗和性能上均已明显优于iptables，所以推荐开启
# 具体原因可参考官网介绍 https://kubernetes.io/zh/blog/2018/07/09/ipvs-based-in-cluster-load-balancing-deep-dive/
yum install ipvsadm ipset sysstat conntrack libseccomp -y
# 加载模块，最后一条4.18及以下内核使用nf_conntrack_ipv4，4.19已改为nf_conntrack
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
# systemd-modules-load加入开机自启
systemctl enable --now systemd-modules-load
# 自定义内核参数优化配置文件
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
# 重启查看IPVS模块是否依旧加载
reboot
lsmod | grep -e ip_vs -e nf_conntrack
```

- 保证每台服务器中IPVS加载成功，以k8s-master01为例，如图：

![20220305153926](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220305153926.png)

### 部署Docker(ALL)

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

### 安装kubernetes(ALL)

一般kubectl在master节点安装即可,node节点装不装均可

```bash
# 查看可以安装的版本号
yum list kubeadm --showduplicates | sort -r
# 不指定版本的话默认安装最新版本安装
yum install -y kubelet kubeadm kubectl
# 指定版本进行安装，如1.22.4
yum install -y kubelet-1.22.4 kubeadm-1.22.4 kubectl-1.22.4
# 配置pause镜像仓库，默认的gcr.io国内无法访问，可以使用阿里仓库
cat << EOF > /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS="--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.5"
EOF
# 先加入开机启动
systemctl enable kubelet
# 先不启动kubelet,因为会启动失败,提示初始化文件不存在,kubernetes集群初始化完成后会启动
```

### 高可用组件安装(Master)

```bash
# 所有master节点安装Keepalived和haproxy
yum install keepalived haproxy -y
# 为所有master节点添加haproxy配置，配置都一样，检查最后三行主机名和IP地址对应上就行
mkdir /etc/haproxy
cat << EOF > /etc/haproxy/haproxy.cfg
global
  maxconn  2000
  ulimit-n  16384
  log  127.0.0.1 local0 err
  stats timeout 30s

defaults
  log global
  mode  http
  option  httplog
  timeout connect 5000
  timeout client  50000
  timeout server  50000
  timeout http-request 15s
  timeout http-keep-alive 15s

frontend monitor-in
  bind *:33305
  mode http
  option httplog
  monitor-uri /monitor

frontend k8s-master
  bind 0.0.0.0:16443
  bind 127.0.0.1:16443
  mode tcp
  option tcplog
  tcp-request inspect-delay 5s
  default_backend k8s-master

backend k8s-master
  mode tcp
  option tcplog
  option tcp-check
  balance roundrobin
  default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
  server k8s-master01  192.168.43.183:6443  check
  server k8s-master02  192.168.43.184:6443  check
  server k8s-master03  192.168.43.185:6443  check
EOF
# keepalived配置不一样，注意区分网卡名、IP地址和虚拟IP地址
# 检查服务器网卡名
ip a 或 ifconfig
# k8s-master01 Keepalived配置
mkdir /etc/keepalived
cat << EOF > /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
    script_user root
    enable_script_security
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 5
    weight -5
    fall 2  
    rise 1
}
vrrp_instance VI_1 {
    state MASTER
    interface ens33
    mcast_src_ip 192.168.43.183
    virtual_router_id 51
    priority 101
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        192.168.43.182
    }
    track_script {
       chk_apiserver
    }
}
EOF
# k8s-master02 Keepalived配置
mkdir /etc/keepalived
cat << EOF > /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
    script_user root
    enable_script_security
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 5
    weight -5
    fall 2  
rise 1
}
vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    mcast_src_ip 192.168.43.184
    virtual_router_id 51
    priority 100
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        192.168.43.182
    }
    track_script {
       chk_apiserver
    }
}
EOF
# k8s-master03 Keepalived配置
mkdir /etc/keepalived
cat << EOF > /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
    script_user root
    enable_script_security
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 5
    weight -5
    fall 2  
rise 1
}
vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    mcast_src_ip 192.168.43.185
    virtual_router_id 51
    priority 100
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        192.168.43.182
    }
    track_script {
       chk_apiserver
    }
}
EOF
# 所有master节点配置Keepalived健康检查脚本
cat << EOF > /etc/keepalived/check_apiserver.sh 
#!/bin/bash
err=0
for k in \$(seq 1 3)
do
    check_code=\$(pgrep haproxy)
    if [[ \$check_code == "" ]]; then
        err=\$(expr $err + 1)
        sleep 1
        continue
    else
        err=0
        break
    fi
done

if [[ \$err != "0" ]]; then
    echo "systemctl stop keepalived"
    /usr/bin/systemctl stop keepalived
    exit 1
else
    exit 0
fi
EOF
# 赋予可执行权限
chmod +x /etc/keepalived/check_apiserver.sh
# 启动haproxy和Keepalived并加入开机启动
systemctl start haproxy
systemctl start keepalived
systemctl enable haproxy
systemctl enable keepalived

# 测试一波
telnet k8s-master-vip 16443
ping k8s-master-vip
```

### 部署k8s-master(k8s-master01)

在k8s-master01节点上执行，个别步骤在所有master节点执行，已另行说明，没说明的均是在k8s-master01执行。

```bash
# 创建初始化文件，注意版本号和IP对应上，
cat << EOF > kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: 7t2weq.bjbawausm0jaxury
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.43.183
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: k8s-master01
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  certSANs:
  - 192.168.43.182
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: 192.168.43.182:16443
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.23.0
networking:
  dnsDomain: cluster.local
  podSubnet: 172.16.0.0/12
  serviceSubnet: 10.96.0.0/12
scheduler: {}
EOF

# 更新初始化文件
kubeadm config migrate --old-config kubeadm-config.yaml --new-config new.yaml
# 将new.yaml复制到其他master节点上
for i in k8s-master02 k8s-master03;do scp new.yaml $i:/root/;done

# 镜像预下载，节省集群初始化的时间（这一步在所有master节点上执行）
kubeadm config images pull --config /root/new.yaml
# 执行结果如下
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.23.0
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.23.0
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.23.0
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.23.0
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.6
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.5.1-0
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:v1.8.6

# master01初始化，初始化完成后，加入其他节点即可
kubeadm init --config /root/new.yaml  --upload-certs
# 初始化如果失败，可用下面命令清除初始化信息，然后再次尝试初始化
kubeadm reset -f ; ipvsadm --clear  ; rm -rf ~/.kube
# 初始化成功类似于下面输出，保存好这些信息
...
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 192.168.43.182:16443 --token 7t2weq.bjbawausm0jaxury \
        --discovery-token-ca-cert-hash sha256:5257d44118ab035adc5af89dd7d5a24ca4c31c33e1918b3453ea9aa32597121b \
        --control-plane --certificate-key 9a2e86718fceba001c96e503e9df47db3a645d4917bf783decaea9c5d0a726ed

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.43.182:16443 --token 7t2weq.bjbawausm0jaxury \
        --discovery-token-ca-cert-hash sha256:5257d44118ab035adc5af89dd7d5a24ca4c31c33e1918b3453ea9aa32597121b

# 按照提示，如果你是普通用户在操作，执行一下下面几条
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
# 如果是root用户在操作初始化，执行下面一条即可
export KUBECONFIG=/etc/kubernetes/admin.conf

# 查看docker镜像,可以看到kube..和etcd等镜像
docker images

# Token过期后生成新的token（没提示过期下面两步就不用管了）
kubeadm token create --print-join-command
# Master需要生成--certificate-key
kubeadm init phase upload-certs  --upload-certs

```

### 其他节点加入集群

其他master节点加入k8s-master01

```bash
# 根据kubeadm init提示的token
kubeadm join 192.168.43.182:16443 --token 7t2weq.bjbawausm0jaxury \
        --discovery-token-ca-cert-hash sha256:5257d44118ab035adc5af89dd7d5a24ca4c31c33e1918b3453ea9aa32597121b \
        --control-plane --certificate-key 9a2e86718fceba001c96e503e9df47db3a645d4917bf783decaea9c5d0a726ed
```

node节点加入k8s-master01

```bash
# 根据kubeadm init提示的token
kubeadm join 192.168.43.182:16443 --token 7t2weq.bjbawausm0jaxury \
        --discovery-token-ca-cert-hash sha256:5257d44118ab035adc5af89dd7d5a24ca4c31c33e1918b3453ea9aa32597121b
```

```bash
# master01上查看加入后的节点信息，因为还未配置CNI插件，所以node之间通信还未打通
[root@k8s-master01 ~]# kubectl get no
NAME           STATUS     ROLES                  AGE     VERSION
k8s-master01   NotReady   control-plane,master   19m     v1.23.0
k8s-master02   NotReady   control-plane,master   3m39s   v1.23.0
k8s-master03   NotReady   control-plane,master   3m35s   v1.23.0
k8s-node01     NotReady   <none>                 2m21s   v1.23.0
k8s-node02     NotReady   <none>                 2m21s   v1.23.0
```

### 配置calico网络

在k8s-master01节点上执行

网络方案也可以选择其他(例如：flannel等)

```bash
# 先把所需的配置文件从GitHub拉下来
git clone https://github.com/deemoprobe/k8s-ha-install.git
# 切换到1.23分支并进入calico文件夹
cd /root/k8s-ha-install && git checkout manual-installation-v1.23.x && cd calico/
# 替换一下POD网段
POD_SUBNET=`cat /etc/kubernetes/manifests/kube-controller-manager.yaml | grep cluster-cidr= | awk -F= '{print $NF}'`
sed -i "s#POD_CIDR#${POD_SUBNET}#g" calico.yaml
# 应用calico插件
kubectl apply -f calico.yaml
# 集群节点均已处于Ready状态
[root@k8s-master01 calico]# kubectl get no
NAME           STATUS   ROLES                  AGE   VERSION
k8s-master01   Ready    control-plane,master   29m   v1.23.0
k8s-master02   Ready    control-plane,master   13m   v1.23.0
k8s-master03   Ready    control-plane,master   13m   v1.23.0
k8s-node01     Ready    <none>                 11m   v1.23.0
k8s-node02     Ready    <none>                 11m   v1.23.0
# 查看calico Pod是否都正常
kubectl get pod -A
# 如果不正常，可以排查一下，一般是镜像拉取问题，多等待几分钟即可，也可以根据报错简单处理一下
kubectl describe pod XXX -n kube-system
# 比如我这里有个pod处于pending状态，查看一下原因
[root@k8s-master01 calico]# kubectl get pod -A
NAMESPACE     NAME                                       READY   STATUS    RESTARTS      AGE
...
kube-system   calico-typha-8445487f56-hx8w9              1/1     Running   0             11m
kube-system   calico-typha-8445487f56-mh6tp              0/1     Pending   0             11m
kube-system   calico-typha-8445487f56-pxthb              1/1     Running   0             11m
...
# 可以看到提示说2个node节点无法提供足量的pod端口分配需求，而且提示master节点设置了污点
[root@k8s-master01 calico]# kubectl describe pod calico-typha-8445487f56-mh6tp -n kube-system
...
Events:
  Type     Reason            Age                    From               Message
  ----     ------            ----                   ----               -------
  Warning  FailedScheduling  11m                    default-scheduler  0/5 nodes are available: 2 node(s) had taint {node.kubernetes.io/not-ready: }, that the pod didn't tolerate, 3 node(s) had taint {node-role.kubernetes.io/master: }, that the pod didn't tolerate.
  Warning  FailedScheduling  10m                    default-scheduler  0/5 nodes are available: 1 node(s) didn't have free ports for the requested pod ports, 1 node(s) had taint {node.kubernetes.io/not-ready: }, that the pod didn't tolerate, 3 node(s) had taint {node-role.kubernetes.io/master: }, that the pod didn't tolerate.
  Warning  FailedScheduling  10m                    default-scheduler  0/5 nodes are available: 2 node(s) didn't have free ports for the requested pod ports, 3 node(s) had taint {node-role.kubernetes.io/master: }, that the pod didn't tolerate.
  Warning  FailedScheduling  8m24s (x1 over 9m24s)  default-scheduler  0/5 nodes are available: 2 node(s) didn't have free ports for the requested pod ports, 3 node(s) had taint {node-role.kubernetes.io/master: }, that the pod didn't tolerate.'
# 确认一下，可以看到三个master节点打上了不可调度pod的污点
[root@k8s-master01 calico]# for i in k8s-master01 k8s-master02 k8s-master03;do kubectl describe node $i | grep -i taint;done
Taints:             node-role.kubernetes.io/master:NoSchedule
Taints:             node-role.kubernetes.io/master:NoSchedule
Taints:             node-role.kubernetes.io/master:NoSchedule
# 由于是练习环境，我就把不可调度的污点取消了，如果是生产环境，建议扩容node工作节点来实现足量的端口分配
[root@k8s-master01 calico]# for i in k8s-master01 k8s-master02 k8s-master03;do kubectl taint node $i node-role.kubernetes.io/master:NoSchedule-;done
node/k8s-master01 untainted
node/k8s-master02 untainted
node/k8s-master03 untainted
# 污点成功取消
[root@k8s-master01 calico]# for i in k8s-master01 k8s-master02 k8s-master03;do kubectl describe node $i | grep -i taint;done
Taints:             <none>
Taints:             <none>
Taints:             <none>
# 再查看刚才处于pending的pod发现已经处于running状态了
[root@k8s-master01 calico]# kubectl get po -A | grep calico-typha-8445487f56-mh6tp
kube-system   calico-typha-8445487f56-mh6tp              1/1     Running   0             23m
```

## 部署Metrics

在新版的Kubernetes中系统资源的采集均使用Metrics-server，可以通过Metrics采集节点和Pod的内存、磁盘、CPU和网络的使用率。

```bash
# 将Master01节点的front-proxy-ca.crt复制到所有Node节点
[root@k8s-master01 calico]# for i in k8s-node01 k8s-node02;do scp /etc/kubernetes/pki/front-proxy-ca.crt $i:/etc/kubernetes/pki/front-proxy-ca.crt;done
front-proxy-ca.crt                                                                                                   100% 1115   593.4KB/s   00:00    
front-proxy-ca.crt                                                                                                   100% 1115     1.4MB/s   00:00  
# 安装metrics server
[root@k8s-master01 calico]# cd /root/k8s-ha-install/kubeadm-metrics-server
[root@k8s-master01 kubeadm-metrics-server]# kubectl apply -f comp.yaml 
serviceaccount/metrics-server created
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
clusterrole.rbac.authorization.k8s.io/system:metrics-server created
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
service/metrics-server created
deployment.apps/metrics-server created
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
# 查看运行状态
[root@k8s-master01 kubeadm-metrics-server]# kubectl get po -A | grep metrics
kube-system   metrics-server-5cf8885b66-2nnb6            1/1     Running   0             68s
# 部署后便可以查看指标了
[root@k8s-master01 kubeadm-metrics-server]# kubectl top node
NAME           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
k8s-master01   177m         8%     1196Mi          64%       
k8s-master02   153m         7%     1101Mi          58%       
k8s-master03   163m         8%     1102Mi          58%       
k8s-node01     88m          4%     848Mi           45%       
k8s-node02     85m          4%     842Mi           45%       
[root@k8s-master01 kubeadm-metrics-server]# kubectl top po
NAME                     CPU(cores)   MEMORY(bytes)   
nginx-85b98978db-7mn6r   0m           3Mi
```

## 部署Dashboard

Dashboard是一个展示Kubernetes集群资源和Pod日志，甚至可以执行容器命令的web控制台。

```bash
# 直接部署即可
[root@k8s-master01 kubeadm-metrics-server]# cd /root/k8s-ha-install/dashboard/
[root@k8s-master01 dashboard]# ls
dashboard-user.yaml  dashboard.yaml
[root@k8s-master01 dashboard]# kubectl apply -f .
serviceaccount/admin-user created
clusterrolebinding.rbac.authorization.k8s.io/admin-user created
namespace/kubernetes-dashboard created
serviceaccount/kubernetes-dashboard created
service/kubernetes-dashboard created
secret/kubernetes-dashboard-certs created
secret/kubernetes-dashboard-csrf created
secret/kubernetes-dashboard-key-holder created
configmap/kubernetes-dashboard-settings created
role.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
deployment.apps/kubernetes-dashboard created
service/dashboard-metrics-scraper created
deployment.apps/dashboard-metrics-scraper created
```

```bash
# 查看dashboard端口，默认是NodePort模式，访问集群内任意节点的31073端口即可
[root@k8s-master01 dashboard]# kubectl get svc -owide -A | grep dash
kubernetes-dashboard   dashboard-metrics-scraper   ClusterIP   10.105.172.8    <none>        8000/TCP                 19m    k8s-app=dashboard-metrics-scraper
kubernetes-dashboard   kubernetes-dashboard        NodePort    10.99.148.159   <none>        443:31073/TCP            19m    k8s-app=kubernetes-dashboard
```

访问dashboard：<https://集群内任意节点IP:31073>

发现提示隐私设置错误的问题，如图：

![20211213153948](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20211213153948.png)

在Chrome浏览器启动参数加入`--test-type --ignore-certificate-errors`，然后再访问就没有这个安全提示了

![20211213154024](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20211213154024.png)

![20211213154133](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20211213154133.png)

```bash
# 获取登陆令牌（token）
[root@k8s-master01 dashboard]# kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
Name:         admin-user-token-mwnfs
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: 29584392-1cbd-4d5c-91af-9dd4703008aa

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1099 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IjRyZlh6Ukxta0FlajlHREF5ei1mdl8tZmR6ekwteV9fVEIwalQtejRwUk0ifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLW13bmZzIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiIyOTU4NDM5Mi0xY2JkLTRkNWMtOTFhZi05ZGQ0NzAzMDA4YWEiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.pMBMkLAP2AoymIXJC7H47IPu3avdBWPYSZfvjRME7lEQAnbe-SM-yrTFGPzcsJQC3O9gPDvXgIZ1x1tQUtQhc_333GtDMj_VL9oEZxYiOdd578CnBiFmF0BWVX06pAzONgKbguamMD8XEPAvKt4mnlDUr7WCeQJZf_juXKdl7ZOBtrM5Zae0UQHFG6juKLmFP-XxIgoDVIPhcxeAH1ktOHM9Fk1M831hywL1SL2OLHiN52wGLT4WuYrP2iUbJkNpt2PYitSp3iNuh7rESL4Ur7lmFQkLZa9e5vNMCc1wTwOAWvaW4P5TbxtfI_ng4NK_avquiXJY-67D77G-8WKzWg
```

![20211213154451](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20211213154451.png)

## 集群优化(可选)

Docker可在`/etc/docker/daemon.json`自定义优化配置，所有配置可见：[官方docker configuration](https://docs.docker.com/engine/reference/commandline/dockerd/#daemon-configuration-file)，docker常用优化配置见下方注释说明。

```bash
# 优化docker配置
# /etc/docker/daemon.json文件，使用时删除注释，因为JSON文件不支持注释
{
  "allow-nondistributable-artifacts": [],
  "api-cors-header": "",
  "authorization-plugins": [],
  "bip": "",
  "bridge": "",
  "cgroup-parent": "",
  "cluster-advertise": "",
  "cluster-store": "",
  "cluster-store-opts": {},
  "containerd": "/run/containerd/containerd.sock",
  "containerd-namespace": "docker",
  "containerd-plugin-namespace": "docker-plugins",
  "data-root": "", # 数据根目录，大量docker镜像可能会占用较大存储，可以设置系统盘外的挂载盘
  "debug": true,
  "default-address-pools": [
    {
      "base": "172.30.0.0/16",
      "size": 24
    },
    {
      "base": "172.31.0.0/16",
      "size": 24
    }
  ],
  "default-cgroupns-mode": "private",
  "default-gateway": "",
  "default-gateway-v6": "",
  "default-runtime": "runc",
  "default-shm-size": "64M",
  "default-ulimits": {
    "nofile": {
      "Hard": 64000,
      "Name": "nofile",
      "Soft": 64000
    }
  },
  "dns": [],
  "dns-opts": [],
  "dns-search": [],
  "exec-opts": [],
  "exec-root": "",
  "experimental": false,
  "features": {},
  "fixed-cidr": "",
  "fixed-cidr-v6": "",
  "group": "",
  "hosts": [],
  "icc": false,
  "init": false,
  "init-path": "/usr/libexec/docker-init",
  "insecure-registries": [],
  "ip": "0.0.0.0",
  "ip-forward": false,
  "ip-masq": false,
  "iptables": false,
  "ip6tables": false,
  "ipv6": false,
  "labels": [],
  "live-restore": true, # docker进程宕机时容器依然保持存活
  "log-driver": "json-file", # 日志格式
  "log-level": "", # 日志级别
  "log-opts": { # 日志优化
    "cache-disabled": "false",
    "cache-max-file": "5",
    "cache-max-size": "20m",
    "cache-compress": "true",
    "env": "os,customer",
    "labels": "somelabel",
    "max-file": "5", # 最大日志数量
    "max-size": "10m" # 保存的最大日志大小
  },
  "max-concurrent-downloads": 3, # pull下载并发数
  "max-concurrent-uploads": 5, # push上传并发数
  "max-download-attempts": 5,
  "mtu": 0,
  "no-new-privileges": false,
  "node-generic-resources": [
    "NVIDIA-GPU=UUID1",
    "NVIDIA-GPU=UUID2"
  ],
  "oom-score-adjust": -500,
  "pidfile": "",
  "raw-logs": false,
  "registry-mirrors": [], # 镜像加速器地址
  "runtimes": {
    "cc-runtime": {
      "path": "/usr/bin/cc-runtime"
    },
    "custom": {
      "path": "/usr/local/bin/my-runc-replacement",
      "runtimeArgs": [
        "--debug"
      ]
    }
  },
  "seccomp-profile": "",
  "selinux-enabled": false,
  "shutdown-timeout": 15,
  "storage-driver": "",
  "storage-opts": [],
  "swarm-default-advertise-addr": "",
  "tls": true,
  "tlscacert": "",
  "tlscert": "",
  "tlskey": "",
  "tlsverify": true,
  "userland-proxy": false,
  "userland-proxy-path": "/usr/libexec/docker-proxy",
  "userns-remap": ""
}

# 设置证书有效期
[root@k8s-master01 ~]# vim /usr/lib/systemd/system/kube-controller-manager.service
... # 加入下面配置
--experimental-cluster-signing-duration=876000h0m0s
...
[root@k8s-master01 ~]# systemctl daemon-reload
[root@k8s-master01 ~]# systemctl restart kube-controller-manager

# kubelet优化加密算法，默认的算法容易被漏洞扫描；增长镜像下载周期，避免有些大镜像未下载完成就被动死亡退出
# --tls-cipher-suites=TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
# --image-pull-progress-deadline=30m
[root@k8s-master01 ~]# vim /etc/systemd/system/kubelet.service.d/10-kubelet.conf
... # 下面这行中KUBELET_EXTRA_ARGS=后加入配置
Environment="KUBELET_EXTRA_ARGS=--tls-cipher-suites=TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 --image-pull-progress-deadline=30m"
...

# 集群配置优化，详见https://kubernetes.io/zh/docs/tasks/administer-cluster/reserve-compute-resources/
[root@k8s-master01 ~]# vim /etc/kubernetes/kubelet-conf.yml
# 文件中添加如下配置
rotateServerCertificates: true
allowedUnsafeSysctls: # 允许在修改内核参数，此操作按情况选择，用不到就不用设置
 - "net.core*"
 - "net.ipv4.*"
kubeReserved: # 为Kubernetes集群守护进程组件预留资源，例如：kubelet、Runtime等
  cpu: "100m"
  memory: 100Mi
  ephemeral-storage: 1Gi
systemReserved: # 为系统守护进程预留资源，例如：sshd、cron等
  cpu: "100m"
  memory: 100Mi
  ephemeral-storage: 1Gi

# 更改kube-proxy模式为ipvs
[root@k8s-master01 dashboard]# kubectl edit cm kube-proxy -n kube-system
mode: "ipvs"
# 更新kube-proxy的pod
[root@k8s-master01 dashboard]# kubectl patch daemonset kube-proxy -p "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"date\":\"`date +'%s'`\"}}}}}" -n kube-system
daemonset.apps/kube-proxy patched
# 验证
[root@k8s-master01 dashboard]# curl 127.0.0.1:10249/proxyMode
ipvs

# 为集群节点打标签，删除标签把 = 换成 - 即可
kubectl label nodes k8s-node01 node-role.kubernetes.io/node=
kubectl label nodes k8s-node02 node-role.kubernetes.io/node=
kubectl label nodes k8s-master01 node-role.kubernetes.io/master=
kubectl label nodes k8s-master02 node-role.kubernetes.io/master=
kubectl label nodes k8s-master03 node-role.kubernetes.io/master=
# 添加标签后查看集群状态
[root@k8s-master01 ~]# kubectl get node
NAME           STATUS   ROLES    AGE    VERSION
k8s-master01   Ready    master   100m   v1.23.0
k8s-master02   Ready    master   100m   v1.23.0
k8s-master03   Ready    master   100m   v1.23.0
k8s-node01     Ready    node     100m   v1.23.0
k8s-node02     Ready    node     100m   v1.23.0
```

生产环境建议ETCD集群和Kubernetes集群分离，而且使用高性能数据盘存储数据，根据情况决定是否将Master节点也作为Pod调度节点。

## 测试集群

```bash
# 测试namespace
kubectl get namespace
kubectl create namespace test
kubectl get namespace
kubectl delete namespace test

# 创建nginx实例并开放端口
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
# 查看调度状态和端口号
[root@k8s-master01 calico]# kubectl get pod,svc -owide
NAME                         READY   STATUS    RESTARTS   AGE     IP              NODE         NOMINATED NODE   READINESS GATES
pod/nginx-85b98978db-7mn6r   1/1     Running   0          2m16s   172.27.14.193   k8s-master02   <none>           <none>

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE    SELECTOR
service/kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP        59m    <none>
service/nginx        NodePort    10.104.33.99   <none>        80:31720/TCP   2m6s   app=nginx
```

可见调度到了k8s-master02（IP地址是192.168.43.184）上，对应的NodePort为31720

在浏览器输入<http://192.168.43.184:31720/> 访问nginx，访问结果如图

![20211213143124](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20211213143124.png)

至此，基于kubeadm的Kubernetes高可用集群搭建成功。
