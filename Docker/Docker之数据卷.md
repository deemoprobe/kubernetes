# Docker之数据卷

主要作用：做持久化，便于数据的保存和共享，类似于Redis中的aof和rdb文件

```shell
# 创建同步数据卷
docker run -it -v /宿主机绝对路径:/容器绝对路径 IMAGE

[root@docker /]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
centos              latest              0d120b6ccaa8        3 months ago        215MB
[root@docker /]# docker run -it -v /myDataVolume:/myContainerVolume centos
# 容器内部自动创建myContainerVolume
[root@36279a03fe2d /]# ls
bin  dev  etc  home  lib  lib64  lost+found  media  mnt  myContainerVolume  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
# 宿主机自动创建myDataVolume
[root@docker /]# ls
bin  boot  dev  etc  home  lib  lib64  media  mnt  myDataVolume  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
# 查看数据卷同步信息
[root@docker /]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
36279a03fe2d        centos              "/bin/bash"         19 minutes ago      Up 6 minutes                            interesting_neumann
[root@docker /]# docker inspect 36279a03fe2d
[
    {
        "Id": "36279a03fe2d3f1d242d95c5cd313b5ad8e29096db37b0005c20e61005f87a2e",
        "Created": "2020-11-25T08:23:08.293514486Z",
        "Path": "/bin/bash",
        ...
        "HostConfig": {
            "Binds": [
                "/myDataVolume:/myContainerVolume"
            ]
            ...
# 在这两个目录下的数据会相互同步更改，并且持久化（即容器停止也不影响数据同步，下次容器重启后数据自动同步）

# 创建数据卷同时指定权限，不指定默认是有读写权限
# 即宿主机和容器均能创建和更改数据
docker run -it -v /宿主机绝对路径:/容器绝对路径:权限 IMAGE
# 例如指定只读权限，宿主机可创建和更改数据
# 但容器内不能创建和更改数据，只能读取
docker run -it -v /宿主机绝对路径:/容器绝对路径:ro IMAGE
```
