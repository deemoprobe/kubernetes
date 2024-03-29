# 云原生之HELM简单使用

HELM官网：<https://helm.sh/zh/> 本文部署`Helm V3`最新版本，V3和V2版本差异较大，建议直接使用V3。

## 概念和版本差异

在Kubernetes中部署应用，我们需要依次部署Deployment/Service等，步骤比较繁琐。HELM应用管理工具应运而生。

Helm本质就是让k8s的应用管理(Deployment、Service等)可配置，能动态生成。通过动态生成K8S资源清单文件(deployment.yaml、service.yaml)。然后kubectl自动调用K8S资源部署。

Helm负责管理Kubernetes应用--Helm Chart，支持发布的版本管理和控制，它包含在 Kubernetes 集群内部运行应用程序，工具或服务所需的所有资源定义。很大程度上简化了Kubernetes应用的部署和管理，Chart易于创建/发布/分享，Helm和Chart的关系类似Yum和RPM的关系（或Apt和dpkg）。

特点：

- 复杂性管理：即使是最复杂的应用，Charts 依然可以描述，提供使用单点授权的可重复安装应用程序
- 易于升级：随时随地升级和自定义Hooks消除升级的痛点
- 分发简单：Charts 很容易在公共或私有化服务器上分发和部署
- 支持回滚：使用 helm rollback 可以轻松回滚到之前的发布版本

V2 和 V3版本的差异：<https://helm.sh/docs/topics/v2_v3_migration/>

- 移除Tiller服务端，采用远程Chart库
- Chart库更新
- 默认不再有`local`和`stable`仓库：需自己配置
- Chart apiVersion 更新为 V2
- 动态链接的Chart依赖关系移动至Chart.yaml(移除requirement.yaml)
- 采用XDG目录规范，移除`helm init`和`helm home`
- 移除`crd-install hook` 和 `test-failure hook注释`
- Helm 3的Go库进行大量更新，不兼容V2
- 版本二进制在 `get.helm.sh`
- 移除/替换/新增的命令
  - delete --> uninstall
  - fetch --> pull
  - 移除init和home
  - 移除reset和serve
  - install: 安装时需要添加应用备注名或用`--generate-name`参数
  - inspect --> show
  - template: `-x/--execute` argument renamed to `-s/--show-only`
  - upgrade: Added argument `--history-max` which limits the maximum number of revisions saved per release (0 for no limit)

## 安装并配置仓库

```bash
# 在官方(https://github.com/helm/helm/releases)下载想要的的版本, 当前(2022-01-21)最新稳定版 V3.7.2
# 解压并配置
[root@demo ~]# tar -zxvf helm-v3.7.2-linux-amd64.tar.gz 
linux-amd64/
linux-amd64/helm
linux-amd64/LICENSE
linux-amd64/README.md
[root@demo ~]# mv linux-amd64/helm /usr/local/bin/helm
[root@demo ~]# helm version
version.BuildInfo{Version:"v3.7.2", GitCommit:"663a896f4a815053445eec4153677ddc24a0a361", GitTreeState:"clean", GoVersion:"go1.16.10"}

# helm repo
Usage:
  helm repo [command]

Available Commands:
  add         add a chart repository
  index       generate an index file given a directory containing packaged charts
  list        list chart repositories
  remove      remove one or more chart repositories
  update      update information of available charts locally from chart repositories

# 添加阿里仓库
[root@demo ~]# helm repo add stable https://apphub.aliyuncs.com
"stable" has been added to your repositories
[root@demo ~]# helm repo ls
NAME    URL                        
stable  https://apphub.aliyuncs.com
# 更新
[root@demo ~]# helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈Happy Helming!⎈
```

## 使用HELM

### 查找chart

```bash
Usage:
  helm search [command]

Available Commands:
  hub         search for charts in the Artifact Hub or your own hub instance
  repo        search repositories for a keyword in charts
```

```bash
# 搜索HELM的远程仓库(Artifact Hub)中的chart，国内访问太慢
[root@demo ~]# helm search hub
# 搜索hub中特定chart
[root@demo ~]# helm search hub | grep nginx
URL                                                     CHART VERSION   APP VERSION     DESCRIPTION                                       
https://hub.helm.sh/charts/wiremind/nginx               2.1.1                           An NGINX HTTP server                              
https://hub.helm.sh/charts/bitnami/nginx                8.4.1           1.19.6          Chart for the nginx server
...
# 搜索本地仓库的chart, 前提是已经配好仓库, 跟上仓库名搜索即可
[root@demo ~]# helm search repo stable
# 搜索本地特定chart
[root@demo ~]# helm search repo stable | grep nginx
stable/nginx                            5.1.5           1.16.1                          Chart for the nginx server 
...
```

### 安装chart

在安装过程中，helm 客户端会打印一些有用的信息，其中包括：哪些资源已经被创建，release当前的状态（Release 是运行在 Kubernetes 集群中的 chart实例），以及你是否还需要执行额外的配置步骤。

Helm按照以下顺序安装资源：

- Namespace
- NetworkPolicy
- ResourceQuota
- LimitRange
- PodSecurityPolicy
- PodDisruptionBudget
- ServiceAccount
- Secret
- SecretList
- ConfigMap
- StorageClass
- PersistentVolume
- PersistentVolumeClaim
- CustomResourceDefinition
- ClusterRole
- ClusterRoleList
- ClusterRoleBinding
- ClusterRoleBindingList
- Role
- RoleList
- RoleBinding
- RoleBindingList
- Service
- DaemonSet
- Pod
- ReplicationController
- ReplicaSet
- Deployment
- HorizontalPodAutoscaler
- StatefulSet
- Job
- CronJob
- Ingress
- APIService

Helm 客户端不会等到所有资源都运行才退出。许多 charts 需要大小超过 600M 的 Docker 镜像，可能需要很长时间才能安装到集群中。可以使用 helm status 来追踪 release 的状态。

```bash
# 用法
To override values in a chart, use either the '--values' flag and pass in a file
or use the '--set' flag and pass configuration from the command line, to force
a string value use '--set-string'. In case a value is large and therefore
you want not to use neither '--values' nor '--set', use '--set-file' to read the
single large value from file.

    $ helm install -f myvalues.yaml myredis ./redis
or
    $ helm install --set name=prod myredis ./redis
or
    $ helm install --set-string long_int=1234567890 myredis ./redis
or
    $ helm install --set-file my_script=dothings.sh myredis ./redis

Usage:
  helm install [NAME] [CHART] [flags]
```

```bash
# 安装stable下的mariadb这个chart包, 发布名配置为adb
[root@demo ~]# helm install adb stable/mariadb
WARNING: This chart is deprecated
NAME: adb
LAST DEPLOYED: Wed Jan 27 09:57:39 2021
NAMESPACE: dev
STATUS: deployed
...
# 查看安装的包
[root@demo ~]# helm ls
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                           APP VERSION
adb             dev             1               2021-01-27 09:57:39.461456432 +0800 CST deployed        mariadb-7.3.14                  10.3.22    
nginx-ingress   dev             1               2021-01-11 13:50:42.238878136 +0800 CST deployed        nginx-ingress-controller-5.3.4  0.29.0 
# 查看adb发布状态
[root@demo ~]# helm status adb
NAME: adb
LAST DEPLOYED: Wed Jan 27 09:57:39 2021
NAMESPACE: dev
STATUS: deployed
REVISION: 1
NOTES:
...
```

### 安装前自定义chart包

```bash
# 查看chart的用法
Usage:
  helm show [command]

Aliases:
  show, inspect

Available Commands:
  all         show all information of the chart
  chart       show the chart's definition
  readme      show the chart's README
  values      show the chart's values
```

```bash
# 查看chart所有信息
[root@demo ~]# helm show all stable/mariadb
# 查看chart的定义
[root@demo ~]# helm show chart stable/mariadb
apiVersion: v1
appVersion: 10.3.22
deprecated: true
description: DEPRECATED Fast, reliable, scalable, and easy to use open-source relational
  database system. MariaDB Server is intended for mission-critical, heavy-load production
  systems as well as for embedding into mass-deployed software. Highly available MariaDB
  cluster.
home: https://mariadb.org
icon: https://bitnami.com/assets/stacks/mariadb/img/mariadb-stack-220x234.png
keywords:
- mariadb
- mysql
- database
- sql
- prometheus
name: mariadb
sources:
- https://github.com/bitnami/bitnami-docker-mariadb
- https://github.com/prometheus/mysqld_exporter
version: 7.3.14
# 查看chart说明文档
[root@demo ~]# helm show readme stable/mariadb
# 查看chart配置项
[root@demo ~]# helm show values stable/mariadb
```

安装前自定义chart配置

```bash
# 先在本地写一个文件
[root@demo ~]# echo '{mariadbUser: user0, mariadbDatabase: user0db}' > config.yaml
# 将文件中的配置导入chart包, --generate-name 随机命名
[root@demo ~]# helm install -f config.yaml stable/mariadb --generate-name
NAME: mariadb-1611714264
LAST DEPLOYED: Wed Jan 27 10:24:30 2021
NAMESPACE: dev
STATUS: deployed
REVISION: 1
NOTES:
...
# 指定名称
[root@demo ~]# helm install -f config.yaml adb1 stable/mariadb
NAME: adb1
LAST DEPLOYED: Wed Jan 27 10:30:09 2021
NAMESPACE: dev
STATUS: deployed
REVISION: 1
...
# 指定namespace安装chart
[root@demo ~]# helm install -f config.yaml adb2 stable/mariadb -n default
NAME: adb2
LAST DEPLOYED: Wed Jan 27 11:08:55 2021
NAMESPACE: default
STATUS: deployed
...
# 查看所有namespace下的chart
[root@demo ~]# helm ls -A
NAME                    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                           APP VERSION
adb                     dev             1               2021-01-27 09:57:39.461456432 +0800 CST deployed        mariadb-7.3.14                  10.3.22    
adb1                    dev             1               2021-01-27 10:30:09.092906006 +0800 CST deployed        mariadb-7.3.14                  10.3.22    
adb2                    default         1               2021-01-27 11:08:55.962934706 +0800 CST deployed        mariadb-7.3.14                  10.3.22    
mariadb-1611714264      dev             1               2021-01-27 10:24:30.470424053 +0800 CST deployed        mariadb-7.3.14                  10.3.22    
metrics-server          kube-system     1               2021-01-12 15:14:57.092114352 +0800 CST deployed        metrics-server-2.11.4           0.3.6      
nginx-ingress           test            1               2021-01-11 10:34:16.589494767 +0800 CST deployed        nginx-ingress-1.41.2            v0.34.1    
nginx-ingress           dev             1               2021-01-11 13:50:42.238878136 +0800 CST deployed        nginx-ingress-controller-5.3.4  0.29.0     
nginxingress            test            1               2021-01-11 10:50:06.395192654 +0800 CST deployed        nginx-ingress-1.30.3            0.28.0     
```

### 更新和回滚

```bash
# 升级(更新)用法
You can specify the '--values'/'-f' flag multiple times. The priority will be given to the
last (right-most) file specified. For example, if both myvalues.yaml and override.yaml
contained a key called 'Test', the value set in override.yaml would take precedence:

    $ helm upgrade -f myvalues.yaml -f override.yaml redis ./redis

You can specify the '--set' flag multiple times. The priority will be given to the
last (right-most) set specified. For example, if both 'bar' and 'newbar' values are
set for a key called 'foo', the 'newbar' value would take precedence:

    $ helm upgrade --set foo=bar --set foo=newbar redis ./redis

Usage:
  helm upgrade [RELEASE] [CHART] [flags]

# 回退用法

To see revision numbers, run 'helm history RELEASE'.

Usage:
  helm rollback <RELEASE> [REVISION] [flags]
```

```bash
# 准备升级(更新)用的yaml文件, 此处更新了mariadbUser: user0-->mariadbUser: user1
[root@demo ~]# cat config_upgrade.yaml 
{mariadbUser: user1, mariadbDatabase: user0db}
# 可以看到REVISION: 2, 已经有两个版本了
[root@demo ~]# helm upgrade -f config_upgrade.yaml adb stable/mariadb
Release "adb" has been upgraded. Happy Helming!
NAME: adb
LAST DEPLOYED: Wed Jan 27 11:19:38 2021
NAMESPACE: dev
STATUS: deployed
REVISION: 2
NOTES:
...
# 数据已更新
[root@demo ~]# helm get values adb
USER-SUPPLIED VALUES:
mariadbDatabase: user0db
mariadbUser: user1

# 查看版本信息
[root@demo ~]# helm history adb
REVISION        UPDATED                         STATUS          CHART           APP VERSION     DESCRIPTION     
1               Wed Jan 27 09:57:39 2021        superseded      mariadb-7.3.14  10.3.22         Install complete
2               Wed Jan 27 11:19:38 2021        deployed        mariadb-7.3.14  10.3.22         Upgrade complete
# 回退到指定版本
[root@demo ~]# helm rollback adb 1
Rollback was a success! Happy Helming!
[root@demo ~]# helm history adb
REVISION        UPDATED                         STATUS          CHART           APP VERSION     DESCRIPTION     
1               Wed Jan 27 09:57:39 2021        superseded      mariadb-7.3.14  10.3.22         Install complete
2               Wed Jan 27 11:19:38 2021        superseded      mariadb-7.3.14  10.3.22         Upgrade complete
3               Wed Jan 27 11:23:42 2021        deployed        mariadb-7.3.14  10.3.22         Rollback to 1 
# 再回退到2版本
[root@demo ~]# helm rollback adb 2
Rollback was a success! Happy Helming!
[root@demo ~]# helm history adb 
REVISION        UPDATED                         STATUS          CHART           APP VERSION     DESCRIPTION     
1               Wed Jan 27 09:57:39 2021        superseded      mariadb-7.3.14  10.3.22         Install complete
2               Wed Jan 27 11:19:38 2021        superseded      mariadb-7.3.14  10.3.22         Upgrade complete
3               Wed Jan 27 11:23:42 2021        superseded      mariadb-7.3.14  10.3.22         Rollback to 1   
4               Wed Jan 27 11:25:25 2021        deployed        mariadb-7.3.14  10.3.22         Rollback to 2   
# 可以看到数据又回来了
[root@demo ~]# helm get values adb
USER-SUPPLIED VALUES:
mariadbDatabase: user0db
mariadbUser: user1
```

## 删除Release

```bash
# 查看Release
[root@demo ~]# helm ls -A
# 清理当前namespace下资源
[root@demo ~]# helm uninstall ingress-nginx
# 清理指定namespace下资源
[root@demo ~]# helm uninstall ingress-nginx -n ingress-nginx
```
