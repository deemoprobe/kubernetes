# 云原生之Kubernetes应用管理工具HELM

HELM官网: <https://helm.sh/zh/> 本文部署`Helm V3`最新版本, V3和V2版本差异较大, 建议直接使用V3.

## 概念和版本差异

单纯在Kubernetes中部署应用, 我们需要依次部署Deployment/Service等, 步骤比较繁琐. HELM应用管理工具应运而生.

Helm本质就是让k8s的应用管理(Deployment、Service等)可配置,能动态生成.通过动态生成K8S资源清单文件(deployment.yaml、service.yaml).然后kubectl自动调用K8S资源部署.

Helm负责管理Kubernetes应用--Helm Chart, 支持发布的版本管理和控制,很大程度上简化了Kubernetes应用的部署和管理, Chart易于创建/发布/分享,Helm和Chart的关系类似Yum和RPM的关系.

特点:

- 复杂性管理: 即使是最复杂的应用,Charts 依然可以描述, 提供使用单点授权的可重复安装应用程序
- 易于升级: 随时随地升级和自定义Hooks消除升级的痛点
- 分发简单: Charts 很容易在公共或私有化服务器上分发和部署
- 支持回滚: 使用 helm rollback 可以轻松回滚到之前的发布版本

V2 和 V3版本的差异: <https://helm.sh/docs/topics/v2_v3_migration/>

- 移除Tiller服务端, 采用远程Chart库
- Chart库更新
- 默认不再有`local`和`stable`仓库: 需自己配置
- Chart apiVersion 更新为 V2
- 动态链接的Chart依赖关系移动至Chart.yaml(移除requirement.yaml)
- 采用XDG目录规范, 移除`helm init`和`helm home`
- 移除`crd-install hook` 和 `test-failure hook注释`
- Helm 3的Go库进行大量更新, 不兼容V2
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

```shell
# 在官方(https://github.com/helm/helm/releases)下载想要的的版本, 当前(2021-01-11)最新稳定版 V3.4.2
# 解压并配置
[root@k8s-master ~]# tar -zxvf helm-v3.4.2-linux-amd64.tar.gz
[root@k8s-master ~]# mv linux-amd64/helm /usr/local/bin/helm
[root@k8s-master ~]# helm version
version.BuildInfo{Version:"v3.4.2", GitCommit:"23dd3af5e19a02d4f4baa5b2f242645a1a3af629", GitTreeState:"clean", GoVersion:"go1.14.13"}
# 添加阿里仓库
[root@k8s-master ~]# helm repo add apphub https://apphub.aliyuncs.com
"apphub" has been added to your repositories
# 添加官方最新的Helm仓库
[root@k8s-master monitor]# helm repo add stable https://charts.helm.sh/stable
"stable" has been added to your repositories
# 刷新一下
[root@k8s-master monitor]# helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "apphub" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈Happy Helming!⎈
# 查看当前仓库
[root@k8s-master monitor]# helm repo list
NAME    URL                          
apphub  https://apphub.aliyuncs.com  
stable  https://charts.helm.sh/stable

# 可用下面命令删除仓库

```
