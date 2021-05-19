# Docker之常用命令

## 1. RTFM

```shell
Usage:  docker [OPTIONS] COMMAND

A self-sufficient runtime for containers

Options:
      --config string      Location of client config files (default "/root/.docker")
  -c, --context string     Name of the context to use to connect to the daemon (overrides DOCKER_HOST env var and default context set with "docker
                           context use")
  -D, --debug              Enable debug mode
  -H, --host list          Daemon socket(s) to connect to
  -l, --log-level string   Set the logging level ("debug"|"info"|"warn"|"error"|"fatal") (default "info")
      --tls                Use TLS; implied by --tlsverify
      --tlscacert string   Trust certs signed only by this CA (default "/root/.docker/ca.pem")
      --tlscert string     Path to TLS certificate file (default "/root/.docker/cert.pem")
      --tlskey string      Path to TLS key file (default "/root/.docker/key.pem")
      --tlsverify          Use TLS and verify the remote
  -v, --version            Print version information and quit

Management Commands:
  app*        Docker App (Docker Inc., v0.9.1-beta3)
  builder     Manage builds
  buildx*     Build with BuildKit (Docker Inc., v0.5.1-docker)
  config      Manage Docker configs
  container   Manage containers
  context     Manage contexts
  image       Manage images
  manifest    Manage Docker image manifests and manifest lists
  network     Manage networks
  node        Manage Swarm nodes
  plugin      Manage plugins
  scan*       Docker Scan (Docker Inc.)
  secret      Manage Docker secrets
  service     Manage services
  stack       Manage Docker stacks
  swarm       Manage Swarm
  system      Manage Docker
  trust       Manage trust on Docker images
  volume      Manage volumes

Commands:
  attach      Attach local standard input, output, and error streams to a running container
  build       Build an image from a Dockerfile
  commit      Create a new image from a container's changes
  cp          Copy files/folders between a container and the local filesystem
  create      Create a new container
  diff        Inspect changes to files or directories on a container's filesystem
  events      Get real time events from the server
  exec        Run a command in a running container
  export      Export a container's filesystem as a tar archive
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

## 2. 镜像命令

- REPOSITORY: 镜像的仓库源
- TAG: 镜像标签
- IMAGE ID: 镜像ID
- CREATED: 镜像已创建时间
- SIZE: 镜像大小

### 2.1. docker image

Usage:  docker images [OPTIONS] [REPOSITORY[:TAG]]

List images

Options:
  -a, --all             Show all images (default hides intermediate images)
      --digests         Show digests
  -f, --filter filter   Filter output based on conditions provided
      --format string   Pretty-print images using a Go template
      --no-trunc        Don't truncate output
  -q, --quiet           Only show image IDs

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

### 2.2. docker search

Usage:  docker search [OPTIONS] [IMAGE]

Search the Docker Hub for images

Options:
  -f, --filter filter   Filter output based on conditions provided
      --format string   Pretty-print search using a Go template
      --limit int       Max number of search results (default 25)
      --no-trunc        Don't truncate output

```shell
# 从Docker Hub上查询已存在镜像
docker search IMAGE
# 根据stars数目来搜索IMAGE
# 查看15星以上的镜像
docker search -f=stars=15 IMAGE
# 搜索100星以上的nginx镜像，并且不切割摘要信息（摘要全部显示）
docker search --no-trunc -f=stars=100 nginx
```

### 2.3. docker pull

Usage:  docker pull [OPTIONS] NAME[:TAG|@DIGEST]

Pull an image or a repository from a registry

Options:
  -a, --all-tags                Download all tagged images in the repository
      --disable-content-trust   Skip image verification (default true)
      --platform string         Set platform if server is multi-platform capable
  -q, --quiet                   Suppress verbose output

```shell
# 从配置好的仓库拉取镜像, 未配置的话默认从Docker Hub上获取
docker pull IMAGE  <==>  docker pull IMAGE:latest
# 拉取指定版本镜像
docker pull IMAGE:TAG
```

### 2.4. docker rmi

Usage:  docker rmi [OPTIONS] IMAGE [IMAGE...]

Remove one or more images

Options:
  -f, --force      Force removal of the image
      --no-prune   Do not delete untagged parents

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

## 3. 容器命令

### 3.1. docker run

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
# 为nginx镜像随机分配端口映射
[root@docker ~]# docker run -it -P nginx:1.18.0
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
# 查看分配的端口
[root@docker ~]# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                   NAMES
144718cdeaa8        nginx:1.18.0        "/docker-entrypoint.…"   13 seconds ago      Up 12 seconds       0.0.0.0:32768->80/tcp   thirsty_maxwell
# 指定端口映射
[root@docker ~]# docker run -it --name=nginx-test -p 8080:80 nginx:1.18.0
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
# 另起一个terminal查看容器信息
[root@docker ~]# docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED          STATUS         PORTS                  NAMES
58485cdda934   nginx:1.18.0   "/docker-entrypoint.…"   10 seconds ago   Up 9 seconds   0.0.0.0:8080->80/tcp   nginx-test
# 访问nginx
[root@docker ~]# curl localhost:8080
...
<title>Welcome to nginx!</title>
...
# 浏览器访问
```

![20201125144623](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20201125144623.png)

### 3.2. docker ps

Usage:  docker ps [OPTIONS]

List containers

Options:
  -a, --all             Show all containers (default shows just running)
  -f, --filter filter   Filter output based on conditions provided
      --format string   Pretty-print containers using a Go template
  -n, --last int        Show n last created containers (includes all states) (default -1)
  -l, --latest          Show the latest created container (includes all states)
      --no-trunc        Don't truncate output
  -q, --quiet           Only display container IDs
  -s, --size            Display total file sizes

```shell
# 查看所有容器（包括已停止的）ID
[root@docker ~]# docker ps -qa
58485cdda934
38ddd005e21c
1827aed9779f
# 查看最近使用的两个容器
[root@docker ~]# docker ps -n 2
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS                     PORTS     NAMES
58485cdda934   nginx:1.18.0   "/docker-entrypoint.…"   4 minutes ago   Exited (0) 3 minutes ago             nginx-test
38ddd005e21c   busybox        "sh"                     7 days ago      Exited (0) 7 days ago                busybox
# 查看最近一次启动的容器
[root@docker ~]# docker ps -l
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS                     PORTS     NAMES
58485cdda934   nginx:1.18.0   "/docker-entrypoint.…"   5 minutes ago   Exited (0) 3 minutes ago             nginx-test
# 查看所有容器的大小
[root@docker ~]# docker ps -a -s
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS                     PORTS     NAMES          SIZE
58485cdda934   nginx:1.18.0   "/docker-entrypoint.…"   6 minutes ago   Exited (0) 5 minutes ago             nginx-test     1.11kB (virtual 133MB)
38ddd005e21c   busybox        "sh"                     7 days ago      Exited (0) 7 days ago                busybox        29B (virtual 1.24MB)
1827aed9779f   hello-world    "/hello"                 7 days ago      Exited (0) 7 days ago                quirky_raman   0B (virtual 13.3kB)
```

### 3.3. 容器启停

```shell
# 启动已停止的容器
docker start 容器名或ID
# 重启容器
docker restart 容器名或ID
# 停止容器
docker stop 容器名或ID
# 强制停止容器
docker kill 容器名或ID
```

### 3.4. 删除容器

```shell
# 删除已停止容器
docker rm 容器名或ID
# 强制删除(若在运行,也会强制停止后删除)
docker rm -f 容器名或ID
# 删除全部容器
docker rm -f $(docker ps -qa)
or
docker ps -qa | xargs docker rm
```

### 3.5. 进入正在运行的容器

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

```shell
# 进入正在运行的容器并启用交互
docker exec -it 容器ID /bin/bash
# 不进入正在运行的容器直接交互,比如查看根目录
docker exec -it 容器ID ls -al /
exit # 退出
```

### 3.6. 退出容器

exit（等价于Ctrl+D） 退出并关闭容器(适用于docker run命令启动的容器, docker exec 进入容器exit退出后不影响容器状态)

一般需要后台运行的容器可以使用-d先后台启动，需要交互时exec进入容器进行交互。

### 3.7. 容器命令高级操作

docker单独启动容器作为守护进程(后台运行), 启动后`docker ps -a`会发现已经退出了  
原因是：docker容器运行机制决定,docker容器后台运行就必须要有一个前台进程,否则会自动退出  
所以要解决这个问题就是将要运行的进程以前台进程的形式运行

```shell
# 启动容器作为守护进程,这样会直接退出
docker run -d 镜像名
# 后台运行并每两秒在前台输出一次hello
docker run -d centos /bin/sh -c "while true;do echo hello;sleep 2;done"
# 查看日志, 列出时间, 动态打印日志, 保留之前num行
docker logs -f -t --tail num 容器ID
# 实例
[root@k8s-master ~]# docker ps | grep centos
2a478637cb41        centos                                              "/bin/sh -c 'while t…"   13 seconds ago      Up 12 seconds                           priceless_shannon
[root@k8s-master ~]# docker logs -t -f --tail 5 2a478637cb41
2020-11-17T06:54:08.810582561Z hello
2020-11-17T06:54:10.809641077Z hello
2020-11-17T06:54:12.818141600Z hello
2020-11-17T06:54:14.827901332Z hello
2020-11-17T06:54:16.827602049Z hello # 下面几行是动态打印输出的
2020-11-17T06:54:18.837423661Z hello
2020-11-17T06:54:20.846410026Z hello
2020-11-17T06:54:22.862299738Z hello
...

# 查看容器内运行的进程
docker top 容器ID

# 容器内传输数据到宿主机
docker cp 容器ID:/path /宿主机path
```

### 3.8. 镜像的定制

```shell
# 如果该容器内部做了更改，提交打包后更改也包含进去，以此完成镜像的定制
docker commit -a="作者名" -m="提交信息" 容器ID 定制后的镜像名
# 启动一个容器，自定义端口映射，基于nginx:1.18.0镜像
[root@docker ~]# docker run -d -p 8080:80 --name=nginx1.18.0 nginx:1.18.0
[root@docker ~]# docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS                  NAMES
c3fd2aed7368   nginx:1.18.0   "/docker-entrypoint.…"   8 seconds ago   Up 8 seconds   0.0.0.0:8080->80/tcp   nginx1.18.0
# 定制新镜像并打上tag为1.18.0
[root@docker ~]# docker commit -a="deemoprobe" -m="8080:80->nginx1.18.0" c3fd2aed7368 deemoprobe/nginx:1.18.0
sha256:b5fd6cb4ca9e4f24f6332f2d8b712df8256f75a6516c7484c966c797b9f81c5e
# 查看新的镜像已生成
[root@docker ~]# docker images deemoprobe/nginx
REPOSITORY         TAG       IMAGE ID       CREATED          SIZE
deemoprobe/nginx   1.18.0    b5fd6cb4ca9e   42 seconds ago   133MB
# 提交到docker hub
# 首先要创建docker hub账户，然后建立一个新仓库
# 登陆docker hub
[root@docker ~]# docker login
...
Login Succeeded
# 推送
[root@docker ~]# docker push deemoprobe/nginx:1.18.0
The push refers to repository [docker.io/deemoprobe/nginx]
4fa6704c8474: Mounted from library/nginx 
4fe7d87c8e14: Mounted from library/nginx 
6fcbf7acaafd: Mounted from library/nginx 
f3fdf88f1cb7: Mounted from library/nginx 
7e718b9c0c8c: Mounted from library/nginx 
1.18.0: digest: sha256:2db445abcd9b126654035448cada7817300d646a27380916a6b6445e8ede699b size: 1362
# docker hub上就能查看到nginx镜像仓库，并且标签为1.18.0
# 拉下来查看
[root@docker ~]# docker pull deemoprobe/nginx:1.18.0
1.18.0: Pulling from deemoprobe/nginx
f7ec5a41d630: Already exists 
0b20d28b5eb3: Already exists 
1576642c9776: Already exists 
c12a848bad84: Already exists 
03f221d9cf00: Already exists 
Digest: sha256:2db445abcd9b126654035448cada7817300d646a27380916a6b6445e8ede699b
Status: Downloaded newer image for deemoprobe/nginx:1.18.0
docker.io/deemoprobe/nginx:1.18.0
[root@docker ~]# docker images deemoprobe/nginx:1.18.0
REPOSITORY         TAG       IMAGE ID       CREATED          SIZE
deemoprobe/nginx   1.18.0    b5fd6cb4ca9e   20 minutes ago   133MB
```

### 3.9. 高级命令

- 查看docker配置信息

```shell
# 查看容器详情信息的某个字段
docker inspect -f "{{ .首字段.子字段 }}" <ContainerNameOrId>
# 查看容器IP地址
[root@docker ~]# docker inspect -f "{{ .NetworkSettings.IPAddress }}" 38798985efb9
172.17.0.2
# 查看容器主机名
[root@docker ~]# docker inspect -f "{{ .Config.Hostname }}" 38798985efb9
38798985efb9
# 查看开放的端口
[root@docker ~]# docker inspect -f "{{ .Config.ExposedPorts }}" 38798985efb9
map[80/tcp:{}]
```

- 查看网络

```shell
# 启动并开放nginx80端口，80端口映射到主机的1234端口
[root@docker ~]# docker run -p 1234:80 -d nginx
03694540d34be5f69d951f15316dbdeae63fdc60a09e1da078273d5e15cb74ff
[root@docker ~]# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                  NAMES
03694540d34b        nginx               "/docker-entrypoint.…"   4 seconds ago       Up 2 seconds        0.0.0.0:1234->80/tcp   nifty_williams
# 查看端口映射关系
[root@docker ~]# docker port 03694540d34b 80
0.0.0.0:1234
# 查看nat规则
[root@docker ~]# iptables -t nat -nL
...
Chain DOCKER (2 references)
target     prot opt source               destination
RETURN     all  --  0.0.0.0/0            0.0.0.0/0
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:1234 to:172.17.0.3:80
...
# 若容器内部访问不了外网，检查ip_forward和SNAT/MASQUERADE
# 开启ip_forward
[root@docker ~]# sysctl net.ipv4.ip_forward=1
net.ipv4.ip_forward = 1
# 查看SNAT/MASQUERADE是否是ACCEPT
[root@docker ~]# iptables -t nat -nL
Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
# 这条规则指定从容器内出来的包都要进行一次地址转换
MASQUERADE  all  --  172.17.0.0/16        0.0.0.0/0
...
```
