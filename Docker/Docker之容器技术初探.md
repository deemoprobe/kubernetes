# Docker之容器技术初探

## 容器和虚拟机

容器技术溯源可以追溯到`chroot`这个Unix/Linux命令，它可以创造出一个与文件系统隔离的环境，这种环境叫做`chroot jail`，这种环境真实存在，但又不会被外部的进程访问，起到了访问隔离的作用。

![zlgs2h3aa6bl-chrootdirectoriescanbecreatedonvariousplacesinthefilesystem](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/zlgs2h3aa6bl-chrootdirectoriescanbecreatedonvariousplacesinthefilesystem.png)

基于这种思想，Linux内核出现了namespace和cgroup等功能。命名空间（namespace）用于隔离各种资源，Linux最新内核提供了8种命名空间：

- PID：隔离进程
- Network：隔离网络设备、堆栈、IP地址和端口
- Mount：隔离文件系统挂载点
- IPC：隔离进程间通信、共享内存和消息队列
- User：隔离用户和用户组
- UTS：隔离主机名和域名
- Cgroup：隔离进程组的资源（CPU/Mem等）
- Time：最新的命名空间，可用于虚拟化系统时钟

Hypervisor是创建和运行虚拟机的管理程序，有两种类型，一种是直接在裸服务器硬件上工作；另一种是在操作系统之上工作。传统虚拟机通常擦用这种技术进行虚拟化。

![Hypervisor](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/Hypervisor.jpg)

如今，虚拟机和容器都能带来很好的隔离效果，相对来说虚拟机会带来一些开销，无论是启动时间、大小还是运行操作系统的资源使用。容器实际上是进程，启动速度更快，占用空间更小。如果需要更为彻底的隔离，虚拟机不失为一种选择。综合考虑开销、部署响应速度和资源利用率，容器技术更适合云原生架构。

![hzehptn3u06b-TraditionalvsVirtualizedvsContainer](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/hzehptn3u06b-TraditionalvsVirtualizedvsContainer.png)

主要对比如下表：

| 对比项   | 容器              | 虚拟机     |
| -------- | ----------------- | ---------- |
| 开机时间 | 秒级              | 分钟级     |
| 运行机制 | container-runtime | hypervisor |
| 内存使用 | 占用很小          | 占用较大   |
| 隔离强度 | 较弱              | 很强       |
| 部署时长 | 很短              | 较长       |
| 使用     | 较为复杂          | 简易       |

相比于系统级虚拟化，容器技术是进程级别的，具有启动快、体积小等优势，为软件开发、应用部署带来了诸多便利。如使用开源容器Docker技术，应用程序打包推送到镜像中心后，使用时拉取直接运行，实现了“一次打包，到处运行”，非常方便、快捷；使用开源容器编排技术K8S能够实现应用程序的弹性扩容和自动化部署，满足企业灵活扩展信息系统的需求。但是，随着Docker和K8S应用的日益广泛和深入，安全问题也越来越凸显。

Docker容器技术应用中可能存在的技术性安全风险分为镜像安全风险、容器虚拟化安全风险、网络安全风险等类型。

Docker Hub中的镜像可由个人开发者上传，其数量丰富、版本多样，但质量参差不齐，甚至存在包含恶意漏洞的恶意镜像，因而可能存在较大的安全风险。具体而言，Docker镜像的安全风险分布在创建过程、获取来源、获取途径等方方面面。

与传统虚拟机相比，Docker容器不拥有独立的资源配置，且没有做到操作系统内核层面的隔离，因此可能存在资源隔离不彻底与资源限制不到位所导致的容器虚拟化安全风险。

网络安全风险是互联网中所有信息系统所面临的重要风险，不论是物理设备还是虚拟机，都存在难以完全规避的网络安全风险问题。而在轻量级虚拟化的容器网络和容器编排环境中，其网络安全风险较传统网络而言更为复杂严峻。

上面主要是Docker容器技术面临的安全风险，当然如今容器运行时已经不止Docker一种（诸如：containerd、CRI-O等），但面临的安全风险是同样的，都需要引起同样的关注和安全风险的评估。

## 镜像

镜像是一种轻量级、独立可执行的软件包，用来打包软件运行环境和基于该环境开发的软件, 包括代码、运行时、库、环境变量和配置文件等。

### 镜像的特点

Docker镜像都是只读的，当容器启动时，一个新的可写层被加载到镜像的顶部。这一层通常被称作为"容器层"，“容器层”之下的都叫"镜像层"。

### UnionFS(联合文件系统)

UnionFS(联合文件系统): Union文件系统(UnionFS)是一种分层、轻量级并且高性能的文件系统，它支持对文件系统的修改作为一次提交来一层层的叠加，同时可以将不同目录挂载到同一个虚拟文件系统下(unite several directories into a single virtual filesystem)Union文件系统是Docker镜像的基础。镜像可以通过分层来进行继承, 基于基础镜像(没有父镜像)， 可以制作各种具体的应用镜像。

特性: 一次同时加载多个文件系统，但从外面看起来，只能看到一个文件系统，联合加载会把各层文件系统叠加起来，这样最终的文件系统会包含所有底层的文件和目录。

### Docker镜像加载原理

docker的镜像实际上由一层一层的文件系统组成，这种层级的文件系统UnionFS。

bootfs(boot file system),主要包含bootloader和kernel，bootloader主要是引导加载kernel，Linux刚启动时会加载bootfs文件系统，在Docker镜像的最底层是bootfs。这一层与我们典型的Linux/Unix系统是一样的, 包含boot加载器和内核。当boot加载完成之后整个内核就都在内存中了，此时内存的使用权已由bootfs转交给内核，此时系统也会卸载bootfs。

rootfs(root file system),在bootfs之上。包含的就是典型Linux系统中的/dev, /proc, /bin, /etc等标准目录和文件。rootfs就是各种不同的操作系统发行版，比如Ubuntu，Centos等等。

![bootfs](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/bootfs.png)

虚拟机的CentOS一般是几个G，docker这里231M

![20220103180429](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220103180429.png)

对于一个精简的OS，rootfs可以很小，只需要包括最基本的命令、工具和程序库就可以了，因为底层直接用Host的kernel，自己只需要提供rootfs就行了。由此可见对于不同的linux发行版，bootfs基本是一致的，rootfs会有差别，因此不同的发行版可以共用bootfs。

### 分层的镜像

以docker pull为例，在下载的过程中可以看到docker的镜像是在一层一层的在下载

![20220103180200](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220103180200.png)

Docker镜像采用分层结构最大的一个好处就是共享资源。比如：有多个镜像都从相同的base镜像构建而来，那么宿主机只需在磁盘上保存一份base镜像,同时内存中也只需加载一份base镜像，就可以为所有容器服务了。而且镜像的每一层都可以被共享。

![dockerfs](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/dockerfs.png)

## Docker部署

[官方参考文档](https://docs.docker.com/)
[个人博客文档](http://www.deemoprobe.com/yunv/kuberneteskubadm/#DockerALL)

> 网络参考文件：[国家保密局-开源容器技术安全分析](http://www.gjbmj.gov.cn/n1/2021/1014/c411145-32253882.html)
