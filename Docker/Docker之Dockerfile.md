# Docker之Dockerfile

Dockerfile是构建新镜像的命令合集文本文件。

## 字段解析

![dockerfs](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/dockerfs.png)

### 简介

```bash
# 主要字段
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
USER [指定构建镜像和运行容器的用户用户组]
ARG [构建镜像时设定的变量]
LABEL [为镜像添加元数据]
```

### 用法

```bash
# 执行任意路径下的dockerfile
docker build -f /path/to/a/Dockerfile
# 执行当前目录下dockerfile,注意最后的点
# -t(tag)打上tag
docker build -t nginx:v1 .
```

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
WORKDIR path
```

- ENV 设置环境变量

```bash
ENV <key> <value>
ENV <key>=<value> ...

# 示例
ENV dir=/webapp
ENV name deemoprobe
```

- VOLUME 容器数据卷,用于数据保存和持久化

```bash
VOLUME ["/path/to/dir"]

# 示例
VOLUME ["/data"]
```

- USER 指定构建镜像和运行容器的用户用户组

```bash
USER user
USER user:group
USER uid
USER uid:gid
USER user:gid
USER uid:group

# 示例
USER www
USER 1080:tomcat
```

- ARG 构建镜像时设定的变量

```bash
# 与 ENV 作用一致。不过作用域不一样。ARG 设置的环境变量仅对 Dockerfile 内有效，也就是说只有 docker build 的过程中有效，构建好的镜像内不存在此环境变量。
ARG <name>[=<default value>]

# 示例
ARG user=www
```

- ONBUILD 当构建一个被继承的Dockerfile时运行命令

```bash
# 子镜像构建时触发命令并执行。就是Dockerfile里用ONBUILD指定的命令，在本次构建镜像（假设镜像名为test）的过程中不会执行。当有新的Dockerfile使用了该镜像（FROM test），这时执行新镜像的Dockerfile构建时候，会执行test镜像中Dockerfile里的ONBUILD指定的命令。
ONBUILD [INSTRUCTION]

# 示例
ONBUILD RUN yum install wget
ONBUILD ADD . /data
```

- LABEL 为镜像添加元数据

```bash
LABEL <key>=<value> <key>=<value> <key>=<value> ...

# 示例
LABEL version="1.0" des="webapp"
```

## 实例1-简单尝试

```bash
[root@demo ~]# docker build --help

Usage:  docker build [OPTIONS] PATH | URL | -

Build an image from a Dockerfile

Options:
      --add-host list           Add a custom host-to-IP mapping (host:ip)
      --build-arg list          Set build-time variables
      --cache-from strings      Images to consider as cache sources
      --cgroup-parent string    Optional parent cgroup for the container
      --compress                Compress the build context using gzip
      --cpu-period int          Limit the CPU CFS (Completely Fair Scheduler) period
      --cpu-quota int           Limit the CPU CFS (Completely Fair Scheduler) quota
  -c, --cpu-shares int          CPU shares (relative weight)
      --cpuset-cpus string      CPUs in which to allow execution (0-3, 0,1)
      --cpuset-mems string      MEMs in which to allow execution (0-3, 0,1)
      --disable-content-trust   Skip image verification (default true)
  -f, --file string             Name of the Dockerfile (Default is 'PATH/Dockerfile')
      --force-rm                Always remove intermediate containers
      --iidfile string          Write the image ID to the file
      --isolation string        Container isolation technology
      --label list              Set metadata for an image
  -m, --memory bytes            Memory limit
      --memory-swap bytes       Swap limit equal to memory plus swap: '-1' to enable unlimited swap
      --network string          Set the networking mode for the RUN instructions during build (default "default")
      --no-cache                Do not use cache when building the image
      --pull                    Always attempt to pull a newer version of the image
  -q, --quiet                   Suppress the build output and print image ID on success
      --rm                      Remove intermediate containers after a successful build (default true)
      --security-opt strings    Security options
      --shm-size bytes          Size of /dev/shm
  -t, --tag list                Name and optionally a tag in the 'name:tag' format
      --target string           Set the target build stage to build.
      --ulimit ulimit           Ulimit options (default [])
# 创建Dockerfile
cat << EOF > Dockerfile 
FROM centos
MAINTAINER deemoprobe<deemoprobe@gmail.com>
RUN yum install -y curl
CMD [ "/bin/echo", "echo test" ]
EOF
# 构建镜像
[root@demo ~]# docker build -t centos:v1 .
Sending build context to Docker daemon  11.78kB
Step 1/4 : FROM centos
 ---> 5d0da3dc9764
Step 2/4 : MAINTAINER deemoprobe<deemoprobe@gmail.com>
 ---> Running in e8a5c0ccb60b
Removing intermediate container e8a5c0ccb60b
 ---> 8e61f6526576
Step 3/4 : RUN yum install -y curl
 ---> Running in a8a4dd323929
CentOS Linux 8 - AppStream                      3.6 MB/s | 8.4 MB     00:02    
CentOS Linux 8 - BaseOS                         4.1 MB/s | 4.6 MB     00:01    
CentOS Linux 8 - Extras                          15 kB/s |  10 kB     00:00    
Package curl-7.61.1-18.el8.x86_64 is already installed.
Dependencies resolved.
================================================================================
 Package               Architecture Version                  Repository    Size
================================================================================
Upgrading:
 curl                  x86_64       7.61.1-22.el8            baseos       351 k
 libcurl-minimal       x86_64       7.61.1-22.el8            baseos       287 k
 openssl-libs          x86_64       1:1.1.1k-5.el8_5         baseos       1.5 M
Installing dependencies:
 openssl               x86_64       1:1.1.1k-5.el8_5         baseos       709 k
Installing weak dependencies:
 openssl-pkcs11        x86_64       0.4.10-2.el8             baseos        66 k

Transaction Summary
================================================================================
Install  2 Packages
Upgrade  3 Packages

Total download size: 2.8 M
Downloading Packages:
(1/5): openssl-pkcs11-0.4.10-2.el8.x86_64.rpm   1.4 MB/s |  66 kB     00:00    
(2/5): curl-7.61.1-22.el8.x86_64.rpm            3.2 MB/s | 351 kB     00:00    
(3/5): libcurl-minimal-7.61.1-22.el8.x86_64.rpm 2.3 MB/s | 287 kB     00:00    
(4/5): openssl-1.1.1k-5.el8_5.x86_64.rpm        3.3 MB/s | 709 kB     00:00    
(5/5): openssl-libs-1.1.1k-5.el8_5.x86_64.rpm   7.3 MB/s | 1.5 MB     00:00    
--------------------------------------------------------------------------------
Total                                           2.3 MB/s | 2.8 MB     00:01     
warning: /var/cache/dnf/baseos-f6a80ba95cf937f2/packages/openssl-1.1.1k-5.el8_5.x86_64.rpm: Header V3 RSA/SHA256 Signature, key ID 8483c65d: NOKEY
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
  Upgrading        : openssl-libs-1:1.1.1k-5.el8_5.x86_64                   1/8 
  Running scriptlet: openssl-libs-1:1.1.1k-5.el8_5.x86_64                   1/8 
  Installing       : openssl-1:1.1.1k-5.el8_5.x86_64                        2/8 
  Installing       : openssl-pkcs11-0.4.10-2.el8.x86_64                     3/8 
  Upgrading        : libcurl-minimal-7.61.1-22.el8.x86_64                   4/8 
  Upgrading        : curl-7.61.1-22.el8.x86_64                              5/8 
  Cleanup          : curl-7.61.1-18.el8.x86_64                              6/8 
  Cleanup          : libcurl-minimal-7.61.1-18.el8.x86_64                   7/8 
  Cleanup          : openssl-libs-1:1.1.1g-15.el8_3.x86_64                  8/8 
  Running scriptlet: openssl-libs-1:1.1.1g-15.el8_3.x86_64                  8/8 
  Verifying        : openssl-1:1.1.1k-5.el8_5.x86_64                        1/8 
  Verifying        : openssl-pkcs11-0.4.10-2.el8.x86_64                     2/8 
  Verifying        : curl-7.61.1-22.el8.x86_64                              3/8 
  Verifying        : curl-7.61.1-18.el8.x86_64                              4/8 
  Verifying        : libcurl-minimal-7.61.1-22.el8.x86_64                   5/8 
  Verifying        : libcurl-minimal-7.61.1-18.el8.x86_64                   6/8 
  Verifying        : openssl-libs-1:1.1.1k-5.el8_5.x86_64                   7/8 
  Verifying        : openssl-libs-1:1.1.1g-15.el8_3.x86_64                  8/8 

Upgraded:
  curl-7.61.1-22.el8.x86_64              libcurl-minimal-7.61.1-22.el8.x86_64  
  openssl-libs-1:1.1.1k-5.el8_5.x86_64  
Installed:
  openssl-1:1.1.1k-5.el8_5.x86_64       openssl-pkcs11-0.4.10-2.el8.x86_64      

Complete!
Removing intermediate container a8a4dd323929
 ---> ac6ce378b2e9
Step 4/4 : CMD [ "/bin/echo", "echo test" ]
 ---> Running in 99b2e5dab373
Removing intermediate container 99b2e5dab373
 ---> f69dda69804d
Successfully built f69dda69804d
Successfully tagged centos:v1
# 查看结果
[root@demo ~]# docker images
REPOSITORY         TAG       IMAGE ID       CREATED              SIZE
centos             v1        f69dda69804d   About a minute ago   278MB
centos             latest    5d0da3dc9764   3 months ago         231MB
...
```

## 实例2-构建Tomcat

```bash
# 准备Dockerfile
[root@demo ~]# vim Dockerfile 
# 基础镜像centos:latest
FROM centos
# 作者签名
MAINTAINER deemoprobe<deemoprobe@gmail.com>
# 拷贝宿主机当前目录下文件
COPY tomcat.txt /usr/local/tomcat8.txt
# 添加Tomcat安装包并解压至/usr/local
ADD apache-tomcat-8.5.53.tar.gz /usr/local
# 添加jdk安装包并解压至/usr/local
ADD jdk-8u271-linux-x64.tar.gz /usr/local
# 安装vim
RUN yum install -y vim
# 设置环境变量
ENV MYPATH /usr/local
# 指定工作目录，使用ENV设定的环境变量
WORKDIR $MYPATH
# 配置JDK环境
ENV JAVA_HOME /usr/local/jdk1.8.0_271
ENV CLASSPATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
ENV CATALINA_HOME /usr/local/apache-tomcat-8.5.53
ENV CATALINA_BASE /usr/local/apache-tomcat-8.5.53
ENV PATH $PATH:$JAVA_HOME/bin:$CATALINA_HOME/lib:$CATALINA_HOME/bin
# 声明端口8080
EXPOSE 8080
# 启动
# ENTRYPOINT [ "/usr/local/apache-tomcat-8.5.53/bin/startup.sh" ]
# CMD [ "/usr/local/apache-tomcat-8.5.53/bin/catalina.sh", "run" ]
CMD /usr/local/apache-tomcat-8.5.53/bin/startup.sh && tail -F /usr/local/apache-tomcat-8.5.53/bin/logs/catalina.out

# 准备必要的文件到当前目录
[root@demo ~]# echo "tomcat" >> tomcat.txt
# 上传Tomcat和jdk安装包
[root@demo ~]# ls
apache-tomcat-8.5.53.tar.gz  Dockerfile  jdk-8u271-linux-x64.tar.gz  tomcat.txt

# 构建镜像
[root@demo ~]# docker build -t mytomcat8 .
Sending build context to Docker daemon  153.5MB
Step 1/15 : FROM centos
 ---> 5d0da3dc9764
Step 2/15 : MAINTAINER deemoprobe<deemoprobe@gmail.com>
 ---> Using cache
 ---> 8e61f6526576
Step 3/15 : COPY tomcat.txt /usr/local/tomcat8.txt
 ---> 0aa997fe3feb
Step 4/15 : ADD apache-tomcat-8.5.53.tar.gz /usr/local
 ---> 4aa8bbb0d500
Step 5/15 : ADD jdk-8u271-linux-x64.tar.gz /usr/local
 ---> e7a3f287f5a4
Step 6/15 : RUN yum install -y vim
 ---> Running in 16c5177f2b6e
CentOS Linux 8 - AppStream                      1.8 MB/s | 8.4 MB     00:04    
CentOS Linux 8 - BaseOS                         2.8 MB/s | 4.6 MB     00:01    
CentOS Linux 8 - Extras                          14 kB/s |  10 kB     00:00    
Dependencies resolved.
================================================================================
 Package             Arch        Version                   Repository      Size
================================================================================
Installing:
 vim-enhanced        x86_64      2:8.0.1763-16.el8         appstream      1.4 M
Installing dependencies:
 gpm-libs            x86_64      1.20.7-17.el8             appstream       39 k
 vim-common          x86_64      2:8.0.1763-16.el8         appstream      6.3 M
 vim-filesystem      noarch      2:8.0.1763-16.el8         appstream       49 k
 which               x86_64      2.21-16.el8               baseos          49 k

Transaction Summary
================================================================================
Install  5 Packages

Total download size: 7.8 M
Installed size: 30 M
Downloading Packages:
(1/5): gpm-libs-1.20.7-17.el8.x86_64.rpm        770 kB/s |  39 kB     00:00    
(2/5): vim-filesystem-8.0.1763-16.el8.noarch.rp 2.6 MB/s |  49 kB     00:00    
(3/5): vim-enhanced-8.0.1763-16.el8.x86_64.rpm  3.5 MB/s | 1.4 MB     00:00    
(4/5): which-2.21-16.el8.x86_64.rpm             156 kB/s |  49 kB     00:00    
(5/5): vim-common-8.0.1763-16.el8.x86_64.rpm    1.9 MB/s | 6.3 MB     00:03    
--------------------------------------------------------------------------------
Total                                           982 kB/s | 7.8 MB     00:08     
warning: /var/cache/dnf/appstream-02e86d1c976ab532/packages/gpm-libs-1.20.7-17.el8.x86_64.rpm: Header V3 RSA/SHA256 Signature, key ID 8483c65d: NOKEY
CentOS Linux 8 - AppStream                      1.6 MB/s | 1.6 kB     00:00    
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
  Installing       : which-2.21-16.el8.x86_64                               1/5 
  Installing       : vim-filesystem-2:8.0.1763-16.el8.noarch                2/5 
  Installing       : vim-common-2:8.0.1763-16.el8.x86_64                    3/5 
  Installing       : gpm-libs-1.20.7-17.el8.x86_64                          4/5 
  Running scriptlet: gpm-libs-1.20.7-17.el8.x86_64                          4/5 
  Installing       : vim-enhanced-2:8.0.1763-16.el8.x86_64                  5/5 
  Running scriptlet: vim-enhanced-2:8.0.1763-16.el8.x86_64                  5/5 
  Running scriptlet: vim-common-2:8.0.1763-16.el8.x86_64                    5/5 
  Verifying        : gpm-libs-1.20.7-17.el8.x86_64                          1/5 
  Verifying        : vim-common-2:8.0.1763-16.el8.x86_64                    2/5 
  Verifying        : vim-enhanced-2:8.0.1763-16.el8.x86_64                  3/5 
  Verifying        : vim-filesystem-2:8.0.1763-16.el8.noarch                4/5 
  Verifying        : which-2.21-16.el8.x86_64                               5/5 

Installed:
  gpm-libs-1.20.7-17.el8.x86_64         vim-common-2:8.0.1763-16.el8.x86_64    
  vim-enhanced-2:8.0.1763-16.el8.x86_64 vim-filesystem-2:8.0.1763-16.el8.noarch
  which-2.21-16.el8.x86_64             

Complete!
Removing intermediate container 16c5177f2b6e
 ---> 9ada4fe5832b
Step 7/15 : ENV MYPATH /usr/local
 ---> Running in 88554e88d245
Removing intermediate container 88554e88d245
 ---> f2bf79afec88
Step 8/15 : WORKDIR $MYPATH
 ---> Running in d947ecd8b25c
Removing intermediate container d947ecd8b25c
 ---> 369f0b0c45cd
Step 9/15 : ENV JAVA_HOME /usr/local/jdk1.8.0_271
 ---> Running in 38ea9e99ec0f
Removing intermediate container 38ea9e99ec0f
 ---> 1df3c7492a3d
Step 10/15 : ENV CLASSPATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
 ---> Running in 6550635643e2
Removing intermediate container 6550635643e2
 ---> 0ed4c030312d
Step 11/15 : ENV CATALINA_HOME /usr/local/apache-tomcat-8.5.53
 ---> Running in c1f2f0792758
Removing intermediate container c1f2f0792758
 ---> 572937ab416d
Step 12/15 : ENV CATALINA_BASE /usr/local/apache-tomcat-8.5.53
 ---> Running in e52340011fa3
Removing intermediate container e52340011fa3
 ---> 7ed4ff5e933c
Step 13/15 : ENV PATH $PATH:$JAVA_HOME/bin:$CATALINA_HOME/lib:$CATALINA_HOME/bin
 ---> Running in ea347876df2a
Removing intermediate container ea347876df2a
 ---> 23c8fe7ef298
Step 14/15 : EXPOSE 8080
 ---> Running in 1be1de06cd27
Removing intermediate container 1be1de06cd27
 ---> dcdc13f49600
Step 15/15 : CMD /usr/local/apache-tomcat-8.5.53/bin/startup.sh && tail -F /usr/local/apache-tomcat-8.5.53/bin/logs/catalina.out
 ---> Running in 2d497da2180b
Removing intermediate container 2d497da2180b
 ---> 8ad117328e6c
Successfully built 8ad117328e6c
Successfully tagged mytomcat8:latest
# 查看结果
[root@demo ~]# docker images
REPOSITORY         TAG       IMAGE ID       CREATED          SIZE
mytomcat8          latest    8ad117328e6c   50 seconds ago   667MB
...

# 运行一个容器
# 如果有读写权限问题可以加上--privileged=true
[root@demo ~]# docker run -d -p 1080:8080 --name myweb -v /root/web:/usr/local/apache-tomcat-8.5.53/webapps/web -v /root/tomcatlog:/usr/local/apache-tomcat-8.5.53/logs --privileged=true mytomcat8:latest
3df6378afa5d4538887677c74e412c1c1d41b7d53bc840002641bf3f65be3a93
[root@demo ~]# docker ps
CONTAINER ID   IMAGE              COMMAND                  CREATED         STATUS         PORTS                                       NAMES
3df6378afa5d   mytomcat8:latest   "/bin/sh -c '/usr/lo…"   3 seconds ago   Up 2 seconds   0.0.0.0:1080->8080/tcp, :::1080->8080/tcp   myweb
# 访问Tomcat首页，直接 curl localhost:1080 可以看到返回Tomcat首页的HTML源码
[root@demo ~]# curl -I localhost:1080
HTTP/1.1 200 
Content-Type: text/html;charset=UTF-8
Transfer-Encoding: chunked
Date: Mon, 10 Jan 2022 10:38:12 GMT
# 查看WORKDIR
[root@demo ~]# docker exec 3df6378afa5d pwd
/usr/local
# 查看JDK版本
[root@demo ~]# docker exec 3df6378afa5d java -version
java version "1.8.0_271"
Java(TM) SE Runtime Environment (build 1.8.0_271-b09)
Java HotSpot(TM) 64-Bit Server VM (build 25.271-b09, mixed mode)

# 创建web项目，发布服务
[root@demo ~]# ls -l
total 149860
-rw-r--r--. 1 root root  10300600 Jan 10  2022 apache-tomcat-8.5.53.tar.gz
-rw-r--r--. 1 root root      1073 Jan 10 18:04 Dockerfile
-rw-r--r--. 1 root root 143142634 Jan 10  2022 jdk-8u271-linux-x64.tar.gz
drwxr-xr-x. 2 root root       197 Jan 10 18:36 tomcatlog
-rw-r--r--. 1 root root         7 Jan 10 18:06 tomcat.txt
drwxr-xr-x. 2 root root         6 Jan 10 18:36 web
[root@demo ~]# cd web
[root@demo web]# vim web.jsp
<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
<html>
  <head>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
    <title>Insert title here</title>
  </head>
  <body>
    -----------welcome------------
    <%="I am in docker tomcat8"%>
    <br>
    <br>
    <% System.out.println("=============docker tomcat8");%>
  </body>
</html>
[root@demo web]# mkdir WEB-INF
[root@demo web]# cd WEB-INF/
[root@demo WEB-INF]# vim web.xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xmlns="http://java.sun.com/xml/ns/javaee"
  xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd"
  id="WebApp_ID" version="2.5">

  <display-name>test-tomcat8</display-name>
</web-app>
# 查看项目结构
[root@demo ~]# yum install tree -y
[root@demo ~]# tree
.
├── apache-tomcat-8.5.53.tar.gz
├── Dockerfile
├── jdk-8u271-linux-x64.tar.gz
├── tomcatlog
│   ├── catalina.2022-01-10.log
│   ├── catalina.out
│   ├── host-manager.2022-01-10.log
│   ├── localhost.2022-01-10.log
│   ├── localhost_access_log.2022-01-10.txt
│   └── manager.2022-01-10.log
├── tomcat.txt
└── web
    ├── WEB-INF
    │   └── web.xml
    └── web.jsp
3 directories, 12 files
# 查看容器内数据卷同步结果
[root@demo ~]# docker ps
CONTAINER ID   IMAGE              COMMAND                  CREATED          STATUS          PORTS                                       NAMES
3df6378afa5d   mytomcat8:latest   "/bin/sh -c '/usr/lo…"   16 minutes ago   Up 16 minutes   0.0.0.0:1080->8080/tcp, :::1080->8080/tcp   myweb
[root@demo ~]# docker exec 3df6378afa5d ls -l /usr/local/apache-tomcat-8.5.53/webapps/web
total 4
drwxr-xr-x. 2 root root  21 Jan 10 10:49 WEB-INF
-rw-r--r--. 1 root root 500 Jan 10 10:48 web.jsp
[root@demo ~]# docker exec 3df6378afa5d ls -l /usr/local/apache-tomcat-8.5.53/logs
total 24
-rw-r-----. 1 root root 7173 Jan 10 10:49 catalina.2022-01-10.log
-rw-r-----. 1 root root 7173 Jan 10 10:49 catalina.out
-rw-r-----. 1 root root    0 Jan 10 10:36 host-manager.2022-01-10.log
-rw-r-----. 1 root root  459 Jan 10 10:36 localhost.2022-01-10.log
-rw-r-----. 1 root root  281 Jan 10 10:40 localhost_access_log.2022-01-10.txt
-rw-r-----. 1 root root    0 Jan 10 10:36 manager.2022-01-10.log
# 重启一下容器
[root@demo ~]# docker restart 3df6378afa5d

# 访问结果
[root@demo ~]# curl localhost:1080/web/web.jsp

<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
<html>
  <head>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
    <title>Insert title here</title>
  </head>
  <body>
    -----------welcome------------
    I am in docker tomcat8
    <br>
    <br>
    
  </body>
</html>

# 查看日志，可以看到访问记录，其他日志文件可以看到Tomcat启动记录等
[root@demo ~]# cd tomcatlog/
[root@demo tomcatlog]# cat localhost_access_log.2022-01-10.txt
172.17.0.1 - - [10/Jan/2022:10:38:12 +0000] "HEAD / HTTP/1.1" 200 -
172.17.0.1 - - [10/Jan/2022:10:38:21 +0000] "GET / HTTP/1.1" 200 11215
172.17.0.1 - - [10/Jan/2022:10:39:49 +0000] "GET / HTTP/1.1" 200 11215
172.17.0.1 - - [10/Jan/2022:10:39:56 +0000] "GET / HTTP/1.1" 200 11215
172.17.0.1 - - [10/Jan/2022:10:58:16 +0000] "GET /web/web.jsp HTTP/1.1" 200 352
```

> 参考文档：[Docker官方镜像库示例](https://github.com/docker-library)
