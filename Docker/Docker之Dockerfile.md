# Docker之Dockerfile

Dockerfile是创建新镜像的命令合集文本文件。

## 字段解析

![dockerfs](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/dockerfs.png)

### 简介

```shell
FROM image[基础镜像,该文件创建新镜像所依赖的镜像]
MAINTAINER user<email>[作者姓名和邮箱]
RUN command[镜像构建时运行的命令]
ADD [文件拷贝进镜像并解压]
COPY [文件拷贝进镜像]
CMD [容器启动时要运行的命令或参数]
ENTRYPOINT [容器启动时要运行的命令]
EXPOSE port[声明端口]
WORKDIR work_directory[进入容器默认进入的目录]
ENV set_env[创建环境变量]
VOLUME [容器数据卷,用于数据保存和持久化]
ONBUILD [当构建一个被继承的Dockerfile时运行命令]
```

### 用法

- FROM 基础镜像

```bash
# 如果不指定版本，默认使用latest
FROM image
FROM image:tag
FROM image@digest

# 示例
FROM nginx:1.18.0
```

- MAINTAINER 作者

```bash
MAINTAINER user
MAINTAINER email
MAINTAINER user<email>

# 示例
MAINTAINER deemo<deemo@gmail.com>
```

- RUN 构建镜像时执行的命令

```bash
# Dockerfile里的指令每执行一次会在镜像文件系统中新建一层，为了避免多层文件造成镜像过大，多条命令写在一个RUN后面
# RUN指令创建的中间镜像会被缓存，并会在下次构建中使用。如果不想使用这些缓存镜像，可以在构建时指定--no-cache参数，如：docker build --no-cache
RUN command
RUN ["<executable>","<param1>","<param2>",...]

# 示例
RUN yum install -y curl
RUN ["./test.php","dev","offline"]  #等价于 RUN ./test.php dev offline
```

- ADD 本地文件拷贝进镜像，tar类型的会自动解压

```bash
ADD <src> <dest>

# 示例
ADD file /dir/ #添加file到/dir/目录
ADD file dir/ #添加file到{WORKDIR}/dir目录
ADD fi* /dir #通配符，添加所有以fi开头的文件到/dir/目录
```

- COPY 本地文件拷贝进镜像，但不会解压

```bash
COPY <src> <dest>
```

- CMD 容器启动时（docker run时）要运行的命令或参数

```bash
# 可以设置多个CMD,但最后一个生效,前面的不生效,也可以被docker run启动容器时后面加的命令替换
CMD ["<executable>","<param1>","<param2>",...] #执行可执行文件
CMD ["<param1>","<param2>",...] #已设置ENTRYPOINT，则调用ENTRYPOINT后添加CMD参数
CMD command param1 param2 ... #执行shell内部命令

# 示例
CMD ["/usr/bin/ls","-al"]
CMD echo "hello"
```

- ENTRYPOINT 容器启动时要运行的命令

```bash
# 类似于CMD指令，但其不会被docker run的命令行参数指定的指令所覆盖
# 存在多个ENTRYPOINT时，仅最后一个生效
ENTRYPOINT ["<executeable>","<param1>","<param2>",...]

# 示例
ENTRYPOINT ["nginx", "-c"] #定参
CMD ["/etc/nginx/nginx.conf"] #变参
```

- EXPOSE 声明容器端口

```bash
# EXPOSE仅是声明端口。要使其可访问，需要在docker run运行容器时通过-p来指定端口映射，或通过-P参数来映射EXPOSE端口
EXPOSE <port> [<port>...]

# 示例
EXPOSE 80
EXPOSE 80 443
```

- WORKDIR 工作目录

```bash

```

ENV set_env[创建环境变量]

```bash

```

VOLUME [容器数据卷,用于数据保存和持久化]

```bash

```

ONBUILD [当构建一个被继承的Dockerfile时运行命令]

```bash

```

LABEL

## 实例1

```shell
# 执行任意路径下的dockerfile
docker build -f /path/to/a/Dockerfile
or
# 执行当前目录下dockerfile,注意最后的点
# -t(tag)打上tag
docker build -t nginx:v1 .
```

## 实例

如果yum安装过慢或者报错，可以先进入centos容器更新一下`yum update`

```shell
[root@docker docker]# vi Dockerfile 
FROM centos
MAINTAINER deemoprobe<deemoprobe@gmail.com>
RUN yum install -y curl
CMD [ "/bin/echo", "echo test" ]
[root@docker docker]# docker build -t centos:v1 .
Sending build context to Docker daemon  6.656kB
Step 1/4 : FROM centos
 ---> 300e315adb2f
Step 2/4 : MAINTAINER deemoprobe<deemoprobe@gmail.com>
 ---> Using cache
 ---> 99f25e5d1553
Step 3/4 : RUN yum install -y curl
 ---> Running in 135926310e91
CentOS Linux 8 - AppStream                      1.1 MB/s | 6.3 MB     00:05    
CentOS Linux 8 - BaseOS                         752 kB/s | 2.3 MB     00:03    
CentOS Linux 8 - Extras                         6.1 kB/s | 9.6 kB     00:01    
Package curl-7.61.1-14.el8.x86_64 is already installed.
Dependencies resolved.
================================================================================
 Package               Architecture Version                  Repository    Size
================================================================================
Upgrading:
 curl                  x86_64       7.61.1-14.el8_3.1        baseos       353 k
 libcurl-minimal       x86_64       7.61.1-14.el8_3.1        baseos       285 k

Transaction Summary
================================================================================
Upgrade  2 Packages

Total download size: 638 k
Downloading Packages:
(1/2): libcurl-minimal-7.61.1-14.el8_3.1.x86_64 306 kB/s | 285 kB     00:00    
(2/2): curl-7.61.1-14.el8_3.1.x86_64.rpm        215 kB/s | 353 kB     00:01    
--------------------------------------------------------------------------------
Total                                           206 kB/s | 638 kB     00:03     
warning: /var/cache/dnf/baseos-f6a80ba95cf937f2/packages/curl-7.61.1-14.el8_3.1.x86_64.rpm: Header V3 RSA/SHA256 Signature, key ID 8483c65d: NOKEY
CentOS Linux 8 - BaseOS                         1.6 MB/s | 1.6 kB     00:00    
Importing GPG key 0x8483C65D:
 Userid     : "CentOS (CentOS Official Signing Key) <security@centos.org>"
 Fingerprint: 99DB 70FA E1D7 CE22 7FB6 4882 05B5 55B3 8483 C65D
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-centosofficial
Key imported successfully
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                        1/1 
  Upgrading        : libcurl-minimal-7.61.1-14.el8_3.1.x86_64               1/4 
  Upgrading        : curl-7.61.1-14.el8_3.1.x86_64                          2/4 
  Cleanup          : curl-7.61.1-14.el8.x86_64                              3/4 
  Cleanup          : libcurl-minimal-7.61.1-14.el8.x86_64                   4/4 
  Running scriptlet: libcurl-minimal-7.61.1-14.el8.x86_64                   4/4 
  Verifying        : curl-7.61.1-14.el8_3.1.x86_64                          1/4 
  Verifying        : curl-7.61.1-14.el8.x86_64                              2/4 
  Verifying        : libcurl-minimal-7.61.1-14.el8_3.1.x86_64               3/4 
  Verifying        : libcurl-minimal-7.61.1-14.el8.x86_64                   4/4 

Upgraded:
  curl-7.61.1-14.el8_3.1.x86_64     libcurl-minimal-7.61.1-14.el8_3.1.x86_64    

Complete!
Removing intermediate container 135926310e91
 ---> 08781cdcd7c5
Step 4/4 : CMD [ "/bin/echo", "echo test" ]
 ---> Running in 44bc393b77b1
Removing intermediate container 44bc393b77b1
 ---> 6856141ae7b7
Successfully built 6856141ae7b7
Successfully tagged centos:v1
[root@docker docker]# docker images
REPOSITORY         TAG       IMAGE ID       CREATED          SIZE
centos             v1        6856141ae7b7   7 seconds ago    243MB
...
centos             latest    300e315adb2f   5 months ago     209MB
```
