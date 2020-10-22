# Kubernetes 学习规划

## 1.容器及编排 （Containers & Orchestration）

目的: 通过kubeadm 在个人的虚拟机环境安装 Kubernetes 集群

参考:

- 安装 Kubeadm  
<https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/>  
- 使用 Kubeadm启动一个kubernetes 集群  
<https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/>  
- 安装网络插件  
<https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#podnetwork>

## 2.Kubernetes 架构

目的: 使用kubectl管理 Kubernetes

参考:

- Kubectl 参考  
<https://kubernetes.io/docs/reference/kubectl/overview/>  
- Kubectl 使用惯例  
<https://kubernetes.io/docs/reference/kubectl/conventions/>  
Kubectl 备忘录  
<https://kubernetes.io/docs/reference/kubectl/cheatsheet/>  
- 查看和查找resources  
<https://kubernetes.io/docs/reference/kubectl/cheatsheet/#viewing-findingresources>  
- 升级 resources  
<https://kubernetes.io/docs/reference/kubectl/cheatsheet/#updating-resources>  
- 管理运行中的Pods  
<https://kubernetes.io/docs/reference/kubectl/cheatsheet/#interacting-withrunning-pods>

## 3.Pods and Configs

目的: Pods 及其配置文件

参考:

- Kubectl 参考  
<https://kubernetes.io/docs/reference/kubectl/overview/>  
- Kubectl 使用惯例  
<https://kubernetes.io/docs/reference/kubectl/conventions/>  
- Pod概述和模板示例  
<https://kubernetes.io/docs/concepts/workloads/pods/podoverview/>

## 4.Controllers 控制器

目的: 练习Deployments 和 ReplicaSets

参考:

- Deployment 说明及示例  
<https://kubernetes.io/docs/concepts/workloads/controllers/deployment/>  
- Controller 更新  
<https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-adeployment>  
<https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#scaling-adeployment>  
- Kubectl 回滚追溯  
<https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-backa-deployment>  
<https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#checkingrollout-history-of-a-deployment>  
<https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#pausingand-resuming-a-deployment>  
- 健康检查功能  
<https://kubernetes.io/docs/tasks/configure-pod-container/configure-livenessreadiness-startup-probes>  
<https://kubernetes.io/docs/tasks/configure-pod-container/configure-livenessreadiness-startup-probes/#define-a-tcp-liveness-probe>  
- Job 说明及示例  
<https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/>

## 5.定义服务和Kube-Proxy (Services & Kube-Proxy)

目的: 练习创建服务

参考:

- 服务  
<https://kubernetes.io/docs/concepts/services-networking/service/>  
- 便签选择和服务  
<https://kubernetes.io/docs/concepts/servicesnetworking/service/#defining-a-service>  

## 6.管理状态（Managing State）

目的: 练习创建存储卷

参考:

- 卷管理  
<https://kubernetes.io/docs/concepts/storage/volumes/>  
<https://kubernetes.io/docs/tasks/configure-pod-container/configurevolume-storage/>  
- 注释管理  
<https://kubernetes.io/docs/concepts/overview/working-withobjects/annotations/>  
- 下载 API  
<https://kubernetes.io/docs/tasks/inject-data-application/downward-apivolume-expose-pod-information/>  
<https://kubernetes.io/docs/tasks/inject-data-application/environmentvariable-expose-pod-information/>  
- 密钥管理  
<https://kubernetes.io/docs/concepts/configuration/secret/>  
<https://kubernetes.io/docs/tasks/inject-data-application/distributecredentials-secure/>  
- ConfigMaps  
<https://kubernetes.io/docs/tasks/configure-pod-container/configure-podconfigmap>

## 7.API 和Pod 的安全

目的: 联系创建命名空间、配额、限制范围和准入控制

参考:

- 命名空间  
<https://kubernetes.io/docs/concepts/overview/working-withobjects/namespaces/>  
<https://kubernetes.io/docs/tasks/administer-cluster/namespaceswalkthrough/>  
- 资源配额  
<https://kubernetes.io/docs/concepts/policy/resource-quotas/>  
<https://kubernetes.io/docs/tasks/administer-cluster/quota-api-object/>  
- 准入控制  
<https://kubernetes.io/docs/reference/access-authn-authz/admissioncontrollers/>  
<https://kubernetes.io/blog/2019/03/21/a-guide-to-kubernetesadmission-controllers/>  
- Pod的安全策略  
<https://kubernetes.io/docs/reference/access-authn-authz/authorization/>  
<https://kubernetes.io/docs/concepts/policy/pod-security-policy/>

## 8.可观测性 （Observability）

目的: 使用 metrics server 配置 HPAs

参考:

- Metrics Server  
<https://kubernetes.io/docs/tasks/debug-application-cluster/resourcemetrics-pipeline/>  
- HPA  
<https://kubernetes.io/docs/tasks/run-application/horizontal-podautoscale/>  
<https://kubernetes.io/docs/tasks/run-application/horizontal-podautoscale-walkthrough/>

## 9.进口流量(Ingress)

目的: 练习使用 ingress controllers

参考:

- 创建 Namespaces  
<https://kubernetes.io/docs/tasks/administer-cluster/namespaceswalkthrough/#create-new-namespaces>  
- 服务账号  
<https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/>  
- 角色  
<https://kubernetes.io/docs/reference/access-authn-authz/rbac/>  
<https://kubernetes.io/docs/reference/access-authn-authz/rbac/#api-overview>  
- 分布式密钥  
<https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/>  
- Ingress 控制器  
<https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/>  
<https://kubernetes.io/docs/concepts/services-networking/ingress/>  
- 部署Ingress (feat. MiniKube, but still applies to regular clusters)  
<https://kubernetes.io/docs/tasks/access-application-cluster/ingressminikube/#enable-the-ingress-controller>  
- 创建service  
<https://kubernetes.io/docs/concepts/workloads/controllers/deployment/>  
- 创建一个Deployment  
<https://kubernetes.io/docs/concepts/services-networking/service/>

## 10.网络策略(Kubernetes Networking)

目的: 练习网络策略

参考:

- 网络策略  
<https://kubernetes.io/docs/concepts/services-networking/networkpolicies/>  
<https://kubernetes.io/blog/2017/10/enforcing-network-policies-inkubernetes/>  
- 网络策略示例  
<https://kubernetes.io/docs/concepts/services-networking/networkpolicies/#the-networkpolicy-resource>

## 11.Etcd 数据库

目的: Etcd 集群的操作练习

参考:

- Kubernetes中使用Etcd  
<https://kubernetes.io/docs/tasks/administer-cluster/configureupgrade-etcd/>  
- 使用etcdctl  
<https://kubernetes.io/docs/tasks/administer-cluster/configureupgrade-etcd/#replacing-a-failed-etcd-member>  
- 备份etcd  
<https://kubernetes.io/docs/tasks/administer-cluster/configureupgrade-etcd/#backing-up>
