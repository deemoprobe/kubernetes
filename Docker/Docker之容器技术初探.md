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

Hypervisor是创建和运行虚拟机的管理程序，有两种类型，一种是直接在裸服务器硬件上工作；另一种是在操作系统之上工作。传统虚拟机通常采用这种技术进行虚拟化。

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

## Docker部署

[官方参考文档](https://docs.docker.com/)
[个人博客文档](http://www.deemoprobe.com/yunv/kuberneteskubadm/#DockerALL)

## 数据卷

Docker 镜像是由多个文件系统（只读层）叠加而成。当我们启动一个容器的时候，Docker 会加载只读镜像层并在其上（镜像栈顶部）添加一个读写层。如果运行中的容器修改了现有的一个已经存在的文件，那该文件将会从读写层下面的只读层复制到读写层，该文件的只读版本仍然存在，只是已经被读写层中该文件的副本所隐藏。当删除Docker容器，并通过该镜像重新启动时，之前的更改将会丢失。

![dockerfs](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/dockerfs.png)

为了能够保存（持久化）数据以及共享容器间的数据，Docker提出了Volume的概念。简单来说，数据卷是存在于一个或多个容器中的特定文件或文件夹，它可以绕过默认的联合文件系统，以正常的文件或者目录的形式存在于宿主机上。其生存周期独立于容器的生存周期。

![dockervolume](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/dockervolume.png)

Docker提供了三种方式将数据从宿主机挂载到容器中：

- volumes: Docker管理宿主机文件系统的一部分，默认位于 var/lib/docker/volumes 目录中，这是最常用的方式。
- bind mounts: 可以存储在宿主机系统的任意位置，但在目录结构不同的操作系统之间不可移植。
- tmpfs: 挂载存储在宿主机系统的内存中，而不会写入宿主机的文件系统。

### docker volume

```bash
[root@demo ~]# docker volume --help

Usage:  docker volume COMMAND

Manage volumes

Commands:
  create      Create a volume
  inspect     Display detailed information on one or more volumes
  ls          List volumes
  prune       Remove all unused local volumes
  rm          Remove one or more volumes

# 创建volume
[root@demo ~]# docker volume create new_volume
new_volume
[root@demo ~]# docker volume ls
DRIVER    VOLUME NAME
local     new_volume
[root@demo ~]# docker volume inspect new_volume
[
    {
        "CreatedAt": "2022-01-10T07:43:43+08:00",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/new_volume/_data",
        "Name": "new_volume",
        "Options": {},
        "Scope": "local"
    }
]
# 在宿主机可以找到对应目录
[root@demo ~]# ls -al /var/lib/docker/volumes/
..
drwx-----x.  3 root root     19 Jan 10 07:43 new_volume

命令行中可以用-v使用数据卷
-v/--volume，由（:）分隔的三个字段组成，卷名:容器路径:选项。选项可以ro/rw。

--mount，由多个键值对组成，由,分隔，每个由一个key=value元组组成。
type，值可以为 bind，volume，tmpfs。
source，对于命名卷，是卷名。对于匿名卷，这个字段被省略。可能被指定为 source 或 src。
destination，文件或目录将被挂载到容器中的路径。可以指定为 destination，dst 或 target。
volume-opt 可以多次指定。

# 挂载数据卷new_volume到容器的/volume目录，创建文件并查看同步效果
# 下面命令等效于 docker run -itd --name mountvol --mount source=new_volume,target=/volume nginx
[root@demo ~]# docker run -itd --name mountvol -v new_volume:/volume nginx
c3450a454f209f30987f60863547c7a5a60d58fffa19faf89c08d0840cb6e1ac
[root@demo ~]# docker ps
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS          PORTS     NAMES
c3450a454f20   nginx     "/docker-entrypoint.…"   22 seconds ago   Up 20 seconds   80/tcp    mountvol
[root@demo ~]# docker exec -it c3450a454f20 /bin/bash
root@c3450a454f20:/# cd /volume/
root@c3450a454f20:/volume# echo volume > testfile
root@c3450a454f20:/volume# ls  
testfile
root@c3450a454f20:/volume# exit
exit
[root@demo ~]# cat /var/lib/docker/volumes/new_volume/_data/testfile 
volume

# 默认数据卷在容器内挂载内容具备读写（rw）权限，指定只读
[root@demo ~]# docker run -itd --name mountvol2 -v new_volume:/volume:ro nginx
440ca257ed6de9b4387dc83d85b67e071ebafdb27ce5b5b2e4faa970c21cbd97
[root@demo _data]# docker exec -it 440ca257ed6d /bin/bash
root@440ca257ed6d:/# cd /volume/
root@440ca257ed6d:/volume# touch file
touch: cannot touch 'file': Read-only file system

# 清理容器和数据卷
[root@demo _data]# docker volume rm new_volume
Error response from daemon: remove new_volume: volume is in use - [c3450a454f209f30987f60863547c7a5a60d58fffa19faf89c08d0840cb6e1ac]
[root@demo _data]# docker ps -a
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS          PORTS     NAMES
c3450a454f20   nginx     "/docker-entrypoint.…"   16 minutes ago   Up 16 minutes   80/tcp    mountvol
[root@demo _data]# docker stop c3450a454f20
c3450a454f20
[root@demo _data]# docker rm c3450a454f20
c3450a454f20
[root@demo _data]# docker volume rm new_volume
new_volume
[root@demo _data]# docker volume ls
DRIVER    VOLUME NAME

# 清除未使用的数据卷
docker volume prune vol_name
```

### 使用主机目录

```bash
# 将主机任意目录挂载到容器作为数据卷，-v参数，如果宿主机没有相关目录，会自动创建
[root@demo ~]# docker run -itd --name web -v /webapp:/usr/share/nginx/html nginx
46b47e81ee8e53687b78aa109389f75135cdb70005934f1e4d49a992e47eb3ff
[root@demo ~]# docker inspect web | grep -e Mounts -A 9
        "Mounts": [
            {
                "Type": "bind",
                "Source": "/webapp",
                "Destination": "/usr/share/nginx/html",
                "Mode": "",
                "RW": true,
                "Propagation": "rprivate"
            }
        ],
# --mount，宿主机目录不存在会报错
[root@demo ~]# docker run -itd --name web2 --mount type=bind,source=/app/webapp,target=/usr/share/nginx/html,readonly nginx
docker: Error response from daemon: invalid mount config for type "bind": bind source path does not exist: /app/webapp.
See 'docker run --help'.
[root@demo ~]# mkdir -p /app/webapp
[root@demo ~]# docker run -itd --name web2 --mount type=bind,source=/app/webapp,target=/usr/share/nginx/html,readonly nginx
2f0f5e1a42bf63995fbe3918842bd541dd173d1b25dfe458e1043591b9d42ef8
```

> 网络参考文件：[国家保密局-开源容器技术安全分析](http://www.gjbmj.gov.cn/n1/2021/1014/c411145-32253882.html)
