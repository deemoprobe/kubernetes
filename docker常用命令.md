# docker常用命令

```shell

# 查看docker版本
docker version

# 查看docker系统信息
docker info

# 查看正在运行的容器 （-a参数获取包括已停止的所有容器）
docker ps

# 查看镜像
docker images

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
