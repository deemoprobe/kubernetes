# Docker之常用命令

本文整理了docker常用的一些命令。包括镜像命令，容器命令，日志查看，容器的高级操作以及从容器传输文件到宿主机。

## 常用

```bash
# 查看docker版本
docker version

# 查看docker系统信息
docker info

# 查看正在运行的容器
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

## 详细用法

```bash
Usage:  docker COMMAND

A self-sufficient runtime for containers

Commands:
  attach      Attach local standard input, output, and error streams to a running container
  build       Build an image from a Dockerfile
  commit      Create a new image from a container changes
  cp          Copy files/folders between a container and the local filesystem
  create      Create a new container
  diff        Inspect changes to files or directories on a container filesystem
  events      Get real time events from the server
  exec        Run a command in a running container
  export      Export a container filesystem as a tar archive
  history     Show the history of an image
  images      List images
  import      Import the contents from a tarball to create a filesystem image
  info        Display system-wide information
  inspect     Return low-level information on Docker objects
  kill        Kill one or more running containers
  load        Load an image from a tar archive or STDIN
  login       Log in to a Docker registry
  logout      Log out from a Docker registry
  logs        Fetch the logs of a container
  pause       Pause all processes within one or more containers
  port        List port mappings or a specific mapping for the container
  ps          List containers
  pull        Pull an image or a repository from a registry
  push        Push an image or a repository to a registry
  rename      Rename a container
  restart     Restart one or more containers
  rm          Remove one or more containers
  rmi         Remove one or more images
  run         Run a command in a new container
  save        Save one or more images to a tar archive (streamed to STDOUT by default)
  search      Search the Docker Hub for images
  start       Start one or more stopped containers
  stats       Display a live stream of container(s) resource usage statistics
  stop        Stop one or more running containers
  tag         Create a tag TARGET_IMAGE that refers to SOURCE_IMAGE
  top         Display the running processes of a container
  unpause     Unpause all processes within one or more containers
  update      Update configuration of one or more containers
  version     Show the Docker version information
  wait        Block until one or more containers stop, then print their exit codes
```

## 镜像命令

- REPOSITORY: 镜像的仓库源
- TAG: 镜像标签
- IMAGE ID: 镜像ID
- CREATED: 镜像已创建时间
- SIZE: 镜像大小

### docker image

```bash
Usage:  docker images [OPTIONS] [REPOSITORY[:TAG]]

List images

Options:
  -a, --all             Show all images (default hides intermediate images)
      --digests         Show digests
  -f, --filter filter   Filter output based on conditions provided
      --format string   Pretty-print images using a Go template
      --no-trunc        Do not truncate output
  -q, --quiet           Only show image IDs

# 查看镜像
[root@demo ~]# docker images
REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
tomcat        latest    fb5657adc892   12 days ago    680MB
hello-world   latest    feb5d9fea6a5   3 months ago   13.3kB
centos        latest    5d0da3dc9764   3 months ago   231MB
# 查看所有镜像
docker images -a
# 查看镜像ID
docker images -q
# 查看所有镜像ID
docker images -qa
```

### docker search

```bash
Usage:  docker search [OPTIONS] [IMAGE]

Search the Docker Hub for images

Options:
  -f, --filter filter   Filter output based on conditions provided
      --format string   Pretty-print search using a Go template
      --limit int       Max number of search results (default 25)
      --no-trunc        Do not truncate output

# 从Docker Hub上查询已存在镜像
[root@demo ~]# docker search nginx
NAME                              DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
nginx                             Official build of Nginx.                        16062     [OK]       
jwilder/nginx-proxy               Automated Nginx reverse proxy for docker con…   2105                 [OK]
richarvey/nginx-php-fpm           Container running Nginx + PHP-FPM capable of…   819                  [OK]
jc21/nginx-proxy-manager          Docker container for managing Nginx proxy ho…   303                  
linuxserver/nginx                 An Nginx container, brought to you by LinuxS…   161                  
tiangolo/nginx-rtmp               Docker image with Nginx using the nginx-rtmp…   148                  [OK]
...
# 根据stars数目来搜索IMAGE
# 查看800星以上的nginx镜像
[root@demo ~]# docker search nginx -f=stars=800
NAME                      DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
nginx                     Official build of Nginx.                        16062     [OK]       
jwilder/nginx-proxy       Automated Nginx reverse proxy for docker con…   2105                 [OK]
richarvey/nginx-php-fpm   Container running Nginx + PHP-FPM capable of…   819                  [OK]
# 搜索800星以上的nginx镜像，并且不切割摘要信息（摘要全部显示）
[root@demo ~]# docker search nginx --no-trunc -f=stars=800
NAME                      DESCRIPTION                                                                      STARS     OFFICIAL   AUTOMATED
nginx                     Official build of Nginx.                                                         16062     [OK]       
jwilder/nginx-proxy       Automated Nginx reverse proxy for docker containers                              2105                 [OK]
richarvey/nginx-php-fpm   Container running Nginx + PHP-FPM capable of pulling application code from git   819                  [OK]
```

### docker pull

```bash
Usage:  docker pull [OPTIONS] NAME[:TAG|@DIGEST]

Pull an image or a repository from a registry

Options:
  -a, --all-tags                Download all tagged images in the repository
      --disable-content-trust   Skip image verification (default true)
      --platform string         Set platform if server is multi-platform capable
  -q, --quiet                   Suppress verbose output

# 从配置好的仓库拉取镜像, 未配置的话默认从Docker Hub上获取
docker pull IMAGE  <==>  docker pull IMAGE:latest
# 拉取指定版本镜像
docker pull IMAGE:TAG
```

### docker rmi

```bash
Usage:  docker rmi [OPTIONS] IMAGE [IMAGE...]

Remove one or more images

Options:
  -f, --force      Force removal of the image
      --no-prune   Do not delete untagged parents

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
[root@demo ~]# docker rmi hello-world
Untagged: hello-world:latest
Untagged: hello-world@sha256:2498fce14358aa50ead0cc6c19990fc6ff866ce72aeb5546e1d59caac3d0d60f
Deleted: sha256:feb5d9fea6a5e9606aa995e879d862b825965ba48de054caab5ef356dc6b3412
Deleted: sha256:e07ee1baac5fae6a26f30cabfe54a36d3402f96afda318fe0a96cec4ca393359
```

## 容器命令

### docker run

```bash
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]

OPTIONS字段说明:

- --name=自定义容器名
- -d  后台运行容器
- -it 新建伪终端交互运行容器
- -P  随机分配端口映射
- -p  指定端口映射, 有以下四种方式
  - ip:hostPort:containerPort
  - ip::containerPort
  - hostPort:containerPort
  - containerPort

# 先获取镜像
[root@demo ~]# docker pull centos
[root@demo ~]# docker images
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
tomcat       latest    fb5657adc892   12 days ago    680MB
centos       latest    5d0da3dc9764   3 months ago   231MB
# 启动运行CentOS容器(本地有该镜像就直接启动, 没有就自动拉取)
[root@demo ~]# docker run -it centos
[root@f75fd428066f /]# cat /etc/redhat-release 
CentOS Linux release 8.4.2105
[root@f75fd428066f /]# exit
[root@demo ~]# docker run -it --name=centos8 centos
[root@dd16aaf5cbcd /]# exit
[root@demo ~]# docker ps -a
CONTAINER ID   IMAGE     COMMAND       CREATED          STATUS                      PORTS     NAMES
dd16aaf5cbcd   centos    "/bin/bash"   58 seconds ago   Exited (0) 52 seconds ago             centos8
f75fd428066f   centos    "/bin/bash"   3 minutes ago    Exited (0) 2 minutes ago              silly_engelbart
# 为nginx镜像随机分配端口映射
[root@demo ~]# docker run -it -P nginx:1.18.0
Unable to find image 'nginx:1.18.0' locally
1.18.0: Pulling from library/nginx
f7ec5a41d630: Pull complete 
0b20d28b5eb3: Pull complete 
1576642c9776: Pull complete 
c12a848bad84: Pull complete 
03f221d9cf00: Pull complete 
Digest: sha256:e90ac5331fe095cea01b121a3627174b2e33e06e83720e9a934c7b8ccc9c55a0
Status: Downloaded newer image for nginx:1.18.0
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for bash scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
# 另起一个终端，查看分配的端口
[root@demo ~]# docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED          STATUS          PORTS                                     NAMES
97c9f5f24db2   nginx:1.18.0   "/docker-entrypoint.…"   42 seconds ago   Up 40 seconds   0.0.0.0:49153->80/tcp, :::49153->80/tcp   jovial_burnell
# 指定端口映射
[root@demo ~]# docker run -it --name=nginx-web -p 8080:80 nginx:1.18.0
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for bash scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
# 另起一个终端查看容器信息
[root@demo ~]# docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED          STATUS          PORTS                                   NAMES
7e243e46f6f6   nginx:1.18.0   "/docker-entrypoint.…"   23 seconds ago   Up 22 seconds   0.0.0.0:8080->80/tcp, :::8080->80/tcp   nginx-web
# 访问nginx
[root@demo ~]# curl localhost:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
# 在第一个终端可看到访问日志
172.17.0.1 - - [04/Jan/2022:01:08:26 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.29.0" "-"

# 浏览器访问
```

![20201125144623](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20201125144623.png)

### docker ps

```bash
Usage:  docker ps [OPTIONS]

List containers

Options:
  -a, --all             Show all containers (default shows just running)
  -f, --filter filter   Filter output based on conditions provided
      --format string   Pretty-print containers using a Go template
  -n, --last int        Show n last created containers (includes all states) (default -1)
  -l, --latest          Show the latest created container (includes all states)
      --no-trunc        Do not truncate output
  -q, --quiet           Only display container IDs
  -s, --size            Display total file sizes

# 查看在运行容器
[root@demo ~]# docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS                                   NAMES
7e243e46f6f6   nginx:1.18.0   "/docker-entrypoint.…"   3 minutes ago   Up 3 minutes   0.0.0.0:8080->80/tcp, :::8080->80/tcp   nginx-web
# 查看在运行容器ID
[root@demo ~]# docker ps -q
7e243e46f6f6
# 查看所有容器（包含已退出的容器）
[root@demo ~]# docker ps -a
CONTAINER ID   IMAGE          COMMAND                  CREATED          STATUS                      PORTS                                   NAMES
7e243e46f6f6   nginx:1.18.0   "/docker-entrypoint.…"   3 minutes ago    Up 3 minutes                0.0.0.0:8080->80/tcp, :::8080->80/tcp   nginx-web
97c9f5f24db2   nginx:1.18.0   "/docker-entrypoint.…"   5 minutes ago    Exited (0) 4 minutes ago                                            jovial_burnell
dd16aaf5cbcd   centos         "/bin/bash"              10 minutes ago   Exited (0) 10 minutes ago                                           centos8
f75fd428066f   centos         "/bin/bash"              12 minutes ago   Exited (0) 11 minutes ago                                           silly_engelbart
# 查看所有容器ID（包含已退出的容器）
[root@demo ~]# docker ps -qa
7e243e46f6f6
97c9f5f24db2
dd16aaf5cbcd
f75fd428066f
# 查看最近使用的两个容器
[root@demo ~]# docker ps -n 2
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS                     PORTS                                   NAMES
7e243e46f6f6   nginx:1.18.0   "/docker-entrypoint.…"   5 minutes ago   Up 5 minutes               0.0.0.0:8080->80/tcp, :::8080->80/tcp   nginx-web
97c9f5f24db2   nginx:1.18.0   "/docker-entrypoint.…"   7 minutes ago   Exited (0) 6 minutes ago                                           jovial_burnell
# 查看最近一次启动的容器
[root@demo ~]# docker ps -l
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS                                   NAMES
7e243e46f6f6   nginx:1.18.0   "/docker-entrypoint.…"   5 minutes ago   Up 5 minutes   0.0.0.0:8080->80/tcp, :::8080->80/tcp   nginx-web
# 查看容器的大小
[root@demo ~]# docker ps -s
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS                                   NAMES       SIZE
7e243e46f6f6   nginx:1.18.0   "/docker-entrypoint.…"   6 minutes ago   Up 6 minutes   0.0.0.0:8080->80/tcp, :::8080->80/tcp   nginx-web   1.12kB (virtual 133MB)
```

### 容器启停

```bash
# 启动已停止的容器
docker start 容器名或ID
# 重启容器
docker restart 容器名或ID
# 停止容器
docker stop 容器名或ID
# 强制停止容器
docker kill 容器名或ID

# 实例
[root@demo ~]# docker ps -a
CONTAINER ID   IMAGE          COMMAND                  CREATED          STATUS                      PORTS     NAMES
7e243e46f6f6   nginx:1.18.0   "/docker-entrypoint.…"   7 minutes ago    Exited (0) 8 seconds ago              nginx-web
97c9f5f24db2   nginx:1.18.0   "/docker-entrypoint.…"   9 minutes ago    Exited (0) 8 minutes ago              jovial_burnell
dd16aaf5cbcd   centos         "/bin/bash"              14 minutes ago   Exited (0) 14 minutes ago             centos8
f75fd428066f   centos         "/bin/bash"              16 minutes ago   Exited (0) 15 minutes ago             silly_engelbart
[root@demo ~]# docker start 7e243e46f6f6
7e243e46f6f6
[root@demo ~]# docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS                                   NAMES
7e243e46f6f6   nginx:1.18.0   "/docker-entrypoint.…"   7 minutes ago   Up 4 seconds   0.0.0.0:8080->80/tcp, :::8080->80/tcp   nginx-web
[root@demo ~]# docker stop nginx-web
nginx-web
[root@demo ~]# docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```

### 删除容器

```bash
# 删除已停止容器
docker rm 容器名或ID
# 强制删除(若在运行,也会强制停止后删除)
docker rm -f 容器名或ID
# 删除全部容器
docker rm -f $(docker ps -qa)
or
docker ps -qa | xargs docker rm

# 实例
[root@demo ~]# docker rm $(docker ps -qa)
7e243e46f6f6
97c9f5f24db2
dd16aaf5cbcd
f75fd428066f
[root@demo ~]# docker ps -qa
```

### 进入正在运行的容器

```bash
Usage:  docker exec [OPTIONS] CONTAINER COMMAND [ARG...]

Run a command in a running container

Options:
  -d, --detach               Detached mode: run command in the background
      --detach-keys string   Override the key sequence for detaching a container
  -e, --env list             Set environment variables
      --env-file list        Read in a file of environment variables
  -i, --interactive          Keep STDIN open even if not attached
      --privileged           Give extended privileges to the command
  -t, --tty                  Allocate a pseudo-TTY
  -u, --user string          Username or UID (format: <name|uid>[:<group|gid>])
  -w, --workdir string       Working directory inside the container

# 进入正在运行的容器并启用交互
docker exec -it 容器ID /bin/bash
# 不进入正在运行的容器直接交互,比如查看根目录
docker exec -it 容器ID ls -al /
exit # 退出

# 实例
# 后台启一个nginx
[root@demo ~]# docker run -itd --name=test_exec nginx
Unable to find image 'nginx:latest' locally
latest: Pulling from library/nginx
a2abf6c4d29d: Pull complete 
a9edb18cadd1: Pull complete 
589b7251471a: Pull complete 
186b1aaa4aa6: Pull complete 
b4df32aa5a72: Pull complete 
a0bcbecc962e: Pull complete 
Digest: sha256:0d17b565c37bcbd895e9d92315a05c1c3c9a29f762b011a10c54a66cd53c9b31
Status: Downloaded newer image for nginx:latest
864a1bdcf178ae110817a3d2f1d9cbf3b4f6d9bba0d7e477b971c403b5281e8a
[root@demo ~]# docker ps
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS          PORTS     NAMES
864a1bdcf178   nginx     "/docker-entrypoint.…"   28 seconds ago   Up 27 seconds   80/tcp    test_exec
# 进入容器
[root@demo ~]# docker exec -it test_exec /bin/bash
root@864a1bdcf178:/# exit
# 不进入容器执行命令，查看/bin目录下文件数量
[root@demo ~]# docker exec -it test_exec ls -al /bin | wc -l
72
```

### 退出容器

exit（等价于Ctrl+D） 退出并关闭容器(适用于docker run命令启动的容器, docker exec 进入容器exit退出后不影响容器状态)

一般需要后台运行的容器可以使用-d先后台启动，需要交互时exec进入容器进行交互。

### 容器命令高级操作

docker单独启动容器作为守护进程(后台运行), 启动后`docker ps -a`会发现已经退出了  
原因是：docker容器运行机制决定,docker容器后台运行就必须要有一个前台进程,否则会自动退出  
所以要解决这个问题就是将要运行的进程以前台进程的形式运行（或者交互模式启动 -itd）

```bash
# 启动容器作为守护进程,这样会直接退出
docker run -d 镜像名
# 后台运行并每两秒在前台输出一次hello
docker run -d centos /bin/sh -c "while true;do echo hello;sleep 2;done"
# 查看日志, 列出时间, 动态打印日志, 保留之前num行
docker logs -f -t --tail num 容器ID
# 实例
# 先删除所有容器
[root@demo ~]# docker rm $(docker ps -qa) -f
f0b8817bb234
864a1bdcf178
# 后台启动一个centos，发现启动后会直接退出
[root@demo ~]# docker run -d centos
acfd747d7ec4a942374bb526d41d072fe4840e3ba3d1255e67bdee5c2399513a
[root@demo ~]# docker ps -a
CONTAINER ID   IMAGE     COMMAND       CREATED         STATUS                     PORTS     NAMES
acfd747d7ec4   centos    "/bin/bash"   6 seconds ago   Exited (0) 6 seconds ago             dreamy_wilbur
# 加入前台进程的在运行
[root@demo ~]# docker run -d centos /bin/sh -c "while true;do echo hello;sleep 2;done"
000da5e0a32581d4c65cb6a64292010f86feceac8ebea3f715f2d972fa7c7065
[root@demo ~]# docker ps -a
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS                      PORTS     NAMES
000da5e0a325   centos    "/bin/sh -c 'while t…"   3 seconds ago    Up 2 seconds                          xenodochial_shamir
acfd747d7ec4   centos    "/bin/bash"              52 seconds ago   Exited (0) 52 seconds ago             dreamy_wilbur
# 每两秒打印一次
[root@demo ~]# docker logs 000da5e0a325
hello
hello
hello
hello
...

# 查看容器内运行的进程
docker top 容器ID
[root@demo ~]# docker top 000da5e0a325
UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD
root                10408               10388               0                   09:33               ?                   00:00:00            /bin/sh -c while true;do echo hello;sleep 2;done
root                10549               10408               0                   09:35               ?                   00:00:00            /usr/bin/coreutils --coreutils-prog-shebang=sleep /usr/bin/sleep 2

# 容器内传输数据到宿主机
docker cp 容器ID:/path /宿主机path
# 进入一个容器，在根目录下创建文件并拷贝到宿主机根目录
[root@demo ~]# docker exec -it 000da5e0a325 /bin/bash
[root@000da5e0a325 /]# echo test >> test.txt
[root@000da5e0a325 /]# cat test.txt 
test
[root@000da5e0a325 /]# exit
exit
[root@demo ~]# docker cp 000da5e0a325:/test.txt /
[root@demo ~]# cat /test.txt 
test
```

### 镜像的定制

```bash
# 如果该容器内部做了更改，提交打包后更改也包含进去，以此完成镜像的定制
docker commit -a="作者名" -m="提交信息" 容器ID 定制后的镜像名
# 启动一个容器，自定义端口映射，基于nginx:1.18.0镜像
[root@demo ~]# docker run -d -p 8080:80 --name=nginx1.18.0 nginx:1.18.0
02c275a9254c7714d8187dc35efabf8859b245272b1ef101c4411b68aa85d9c3
[root@demo ~]# docker ps --no-trunc
CONTAINER ID                                                       IMAGE          COMMAND                                                CREATED          STATUS          PORTS                                   NAMES
02c275a9254c7714d8187dc35efabf8859b245272b1ef101c4411b68aa85d9c3   nginx:1.18.0   "/docker-entrypoint.sh nginx -g 'daemon off;'"         29 seconds ago   Up 28 seconds   0.0.0.0:8080->80/tcp, :::8080->80/tcp   nginx1.18.0
# 定制新镜像并打上tag为1.18.0
[root@demo ~]# docker commit -a="deemoprobe" -m="nginx:1.18.0 8080->80" 02c275a9254c7714d8187dc35efabf8859b245272b1ef101c4411b68aa85d9c3 deemoprobe/nginx:1.18.0
sha256:00b7979f0210c4ebde226fb86d789a4caef7276b2d92304b3942ef34ce733a96
# 查看新的镜像已生成
[root@demo ~]# docker images | grep deemoprobe
deemoprobe/nginx   1.18.0    00b7979f0210   28 seconds ago   133MB
# 提交到docker hub
# 首先要创建docker hub账户，然后建立一个新仓库
# 登陆docker hub
[root@demo ~]# docker login
...
Login Succeeded
# 推送
[root@demo ~]# docker push deemoprobe/nginx:1.18.0
The push refers to repository [docker.io/deemoprobe/nginx]
4fa6704c8474: Mounted from library/nginx 
4fe7d87c8e14: Mounted from library/nginx 
6fcbf7acaafd: Mounted from library/nginx 
f3fdf88f1cb7: Mounted from library/nginx 
7e718b9c0c8c: Mounted from library/nginx 
1.18.0: digest: sha256:2db445abcd9b126654035448cada7817300d646a27380916a6b6445e8ede699b size: 1362
# docker hub上就能查看到nginx镜像仓库，并且标签为1.18.0
# 拉下来查看
[root@demo ~]# docker pull deemoprobe/nginx:1.18.0
1.18.0: Pulling from deemoprobe/nginx
f7ec5a41d630: Already exists 
0b20d28b5eb3: Already exists 
1576642c9776: Already exists 
c12a848bad84: Already exists 
03f221d9cf00: Already exists 
Digest: sha256:2db445abcd9b126654035448cada7817300d646a27380916a6b6445e8ede699b
Status: Downloaded newer image for deemoprobe/nginx:1.18.0
docker.io/deemoprobe/nginx:1.18.0
[root@demo ~]# docker images deemoprobe/nginx:1.18.0
REPOSITORY         TAG       IMAGE ID       CREATED          SIZE
deemoprobe/nginx   1.18.0    b5fd6cb4ca9e   20 minutes ago   133MB
```

### 高级命令

- 查看docker配置信息

```bash
# 查看容器详情信息的某个字段
docker inspect -f "{{ .首字段.子字段 }}" <ContainerNameOrId>
# 查看容器IP地址
[root@demo ~]# docker inspect -f "{{ .NetworkSettings.IPAddress }}" 38798985efb9
172.17.0.2
# 查看容器主机名
[root@demo ~]# docker inspect -f "{{ .Config.Hostname }}" 38798985efb9
38798985efb9
# 查看开放的端口
[root@demo ~]# docker inspect -f "{{ .Config.ExposedPorts }}" 38798985efb9
map[80/tcp:{}]
```

- 查看网络

```bash
# 启动并开放nginx80端口，80端口映射到主机的1234端口
[root@demo ~]# docker run -p 1234:80 -d nginx
03694540d34be5f69d951f15316dbdeae63fdc60a09e1da078273d5e15cb74ff
[root@demo ~]# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                  NAMES
03694540d34b        nginx               "/docker-entrypoint.…"   4 seconds ago       Up 2 seconds        0.0.0.0:1234->80/tcp   nifty_williams
# 查看端口映射关系
[root@demo ~]# docker port 03694540d34b 80
0.0.0.0:1234
# 查看nat规则
[root@demo ~]# iptables -t nat -nL
...
Chain DOCKER (2 references)
target     prot opt source               destination
RETURN     all  --  0.0.0.0/0            0.0.0.0/0
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:1234 to:172.17.0.3:80
...
# 若容器内部访问不了外网，检查ip_forward和SNAT/MASQUERADE
# 开启ip_forward
[root@demo ~]# sysctl net.ipv4.ip_forward=1
net.ipv4.ip_forward = 1
# 查看SNAT/MASQUERADE是否是ACCEPT
[root@demo ~]# iptables -t nat -nL
Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
# 这条规则指定从容器内出来的包都要进行一次地址转换
MASQUERADE  all  --  172.17.0.0/16        0.0.0.0/0
...
```
