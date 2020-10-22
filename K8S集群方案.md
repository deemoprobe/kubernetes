# Kubernetes 集群方案

## 资源分配

### 节点准备

| 主机节点名称 |      IP      | CPU核心数 | 内存大小 | 磁盘大小 | 系统版本 |
| :----------: | :----------: | :-------: | :------: | :------: | :------: |
|  k8s-master  | 192.168.3.13 |     2     |    4G    |   150G   | centos 8 |
|  k8s-node1   | 192.168.3.14 |     2     |    3G    |   100G   | centos 8 |
|  k8s-node2   | 192.168.3.15 |     2     |    3G    |   100G   | centos 8 |

### Harbor

镜像仓库

### Router软路由

koolshare

### 初始化方案

kubeadm

## 操作步骤

sudo swapoff -a
sudo vi /etc/selinux/config
    selinux=disable
sudo vi /etc/hosts
sudo systemctl stop firewalld
sudo systemctl disable firewalld

kubeadm reset --force

kubeadm init --kubernetes-version=1.19.3  \
--apiserver-advertise-address=192.168.3.13   \
--image-repository registry.aliyuncs.com/google_containers  \
--service-cidr=192.168.0.0/16 \
--pod-network-cidr=10.122.0.0/16

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubeadm join 192.168.3.13:6443 --token vmbhd0.e4cszyn6ozqv9tet \
    --discovery-token-ca-cert-hash sha256:c8fd072fd90c9ddbae3b3945ba0120abf5ed6777de3354a6eb4814a93ed1b844
