# docker常用命令

## 基础

```shell
# 查看docker版本
docker version

# 查看docker系统信息
docker info

# 查看正在运行的容器 （-a参数获取包括已停止的所有容器）
docker ps

# 实时监控容器的运行情况
docker stats

# 查看容器或镜像的底层信息
docker inspect ID

# 查看容器中进程情况
docker top ID

# 查看容器中进程的日志
docker logs ID

# 进入某个容器系统
docker exec -it ID bash

```

## 镜像命令

- REPOSITORY: 镜像的仓库源
- TAG: 镜像标签
- IMAGE ID: 镜像ID
- CREATED: 镜像已创建时间
- SIZE: 镜像大小

### docker image

```shell
# 查看镜像
docker images
# 查看所有镜像
docker images -a
# 查看镜像ID
docker images -q
# 查看所有镜像ID
docker images -qa
```

### docker search

```shell
# 从Docker Hub上查询已存在镜像
docker search IMAGE
# 根据热度(stars)获取排名前10的IMAGE
docker search -s 10 IMAGE
# 根据stars数目来搜索IMAGE
# 查看15星以上的镜像
docker search --filter=stars=15 IMAGE
```

### docker pull

```shell
# 从配置好的仓库拉取镜像, 未配置的话默认从Docker Hub上获取
docker pull IMAGE  <==>  docker pull IMAGE:latest
# 拉取指定版本镜像
docker pull IMAGE:TAG
```

### docker rmi

```shell
# 删除最新版本镜像
docker rmi IMAGE  <==>  docker rmi IMAGE:latest
# 删除指定版本镜像
docker rmi IMAGE:TAG
# -f 强制删除镜像(可删除多层镜像)
docker rmi -f IMAGE
# 指定镜像ID进行删除
docker rmi -f IMAGE_ID
# 删除多个镜像
docker rmi -f IMAGE1 IMAGE2 IMAGE3
# 删除所有镜像
docker rmi -f $(docker images -qa)

# 实例
[root@k8s-master /]# docker rmi hello-world
Error response from daemon: conflict: unable to remove repository reference "hello-world" (must force) - container 01757e7b509e is using its referenced image bf756fb1ae65
# 镜像内部是嵌套关系, 一个镜像可能有多层关系
[root@k8s-master /]# docker rmi -f hello-world
Untagged: hello-world:latest
Untagged: hello-world@sha256:8c5aeeb6a5f3ba4883347d3747a7249f491766ca1caa47e5da5dfcf6b9b717c0
Deleted: sha256:bf756fb1ae65adf866bd8c456593cd24beb6a0a061dedf42b26a993176745f6b
```

## 容器命令

### docker run

docker run [OPTIONS] IMAGE_ID [COMAND] [ARG...]

OPTIONS字段说明:

- --name=自定义容器名
- -d 后台运行容器
- -it 新建伪终端交互运行容器
- -P 分配端口映射
- -p 指定端口映射, 有以下四种方式
  - ip:hostPort:containerPort
  - ip::containerPort
  - hostPort:containerPort
  - containerPort

```shell
# 先获取镜像
[root@k8s-master ~]# docker pull centos
[root@k8s-master ~]# docker images
REPOSITORY                                                        TAG                 IMAGE ID            CREATED             SIZE
centos                                                            latest              0d120b6ccaa8        3 months ago        215MB
# 启动运行CentOS容器(本地有该镜像就直接启动, 没有就自动拉取)
[root@k8s-master ~]# docker run -it 0d120b6ccaa8
[root@5ffb334fc398 /]#
exit
[root@k8s-master ~]# docker run -it --name=mycentos01 0d120b6ccaa8
[root@k8s-master ~]# docker ps
CONTAINER ID        IMAGE                                               COMMAND                  CREATED             STATUS              PORTS               NAMES
7869f8b3be3f        0d120b6ccaa8                                        "/bin/bash"              18 seconds ago      Up 17 seconds                           mycentos01
```

### docker ps

docker ps [OPTIONS]

OPTIONS字段说明:

- -a 列出所有正在运行的容器和历史上运行过的容器
- -l 显示最近运行过的容器
- -n [num] 显示最近创建的num个容器
- -q 显示正在运行容器的ID

### 退出容器

- exit 退出并关闭容器
- Ctrl+P+Q 退出但不关闭容器
