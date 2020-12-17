# 手动升级kubernetes集群

手动升级的还没有详细的方案，大多是基于管理工具部署和升级，比如juju、kubeadm、kops、kubespray等。

## 升级步骤

> **注意：**该升级步骤是实验性的，建议在测试集群上使用，无法保证线上服务不中断，实际升级完成后无需对线上服务做任何操作。

大体上的升级步骤是，先升级master节点，然后再一次升级每台node节点。

## 升级建议

主要包括以下建议：

- 应用使用高级对象定义，如支持滚动更新的`Deployment`对象
- 应用要部署成多个实例
- 使用pod的preStop hook，加强pod的生命周期管理
- 使用就绪和健康检查探针来确保应用存活和及时阻拦应用流量的分发

### 准备

1. 备份kubernetes原先的二进制文件和配置文件。
2. 下载最新版本的kubernetes二进制包，如1.8.5版本，下载二进制包，我们使用的是[kubernetes-server-linux-amd64.tar.gz](https://dl.k8s.io/v1.8.5/kubernetes-server-linux-amd64.tar.gz)，分发到集群的每个节点上。

### 升级master节点

停止master节点的进程

```bash
systemctl stop kube-apiserver
systemctl stop kube-scheduler
systemctl stop kube-controller-manager
systemctl stop kube-proxy
systemctl stop kubelet
```

使用新版本的kubernetes二进制文件替换原来老版本的文件，然后启动master节点上的进程：

```bash
systemctl start kube-apiserver
systemctl start kube-scheduler
systemctl start kube-controller-manager
```

因为我们的master节点同时也作为node节点，所有还要执行下面的”升级node节点“中的步骤。

### 升级node节点

关闭swap

```bash
# 临时关闭
swapoff -a

# 永久关闭，注释掉swap分区即可
vim /etc/fstab
#UUID=65c9f92d-4828-4d46-bf19-fb78a38d2fd1 swap                    swap    defaults        0 0
```

修改kubelet的配置文件

将kubelet的配置文件`/etc/kubernetes/kublet`配置文件中的`KUBELET_API_SERVER="--api-servers=http://172.20.0.113:8080"`行注释掉。

> **注意：**：kubernetes1.7及以上版本已经没有该配置了，API server的地址写在了kubeconfig文件中。

停止node节点上的kubernetes进程：

```bash
systemctl stop kubelet
systemctl stop kube-proxy
```

使用新版本的kubernetes二进制文件替换原来老版本的文件，然后启动node节点上的进程：

```bash
systemctl start kubelet
systemctl start kube-proxy
```

启动新版本的kube-proxy报错找不到`conntrack`命令，使用`yum install -y conntrack-tools`命令安装后重启kube-proxy即可。

## 检查

到此升级完成，在master节点上检查节点状态：

```bash
NAME           STATUS    ROLES     AGE       VERSION
172.20.0.113   Ready     <none>    244d      v1.8.5
172.20.0.114   Ready     <none>    244d      v1.8.5
172.20.0.115   Ready     <none>    244d      v1.8.5
```

所有节点的状态都正常，再检查下原先的运行在kubernetes之上的服务是否正常，如果服务正常的话说明这次升级无误。
