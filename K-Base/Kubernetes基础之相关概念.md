# Kubernetes基础之相关概念

## 1. Kubernetes是什么

Kubernetes 是开源的生产级别容器编排引擎，实现容器的自动发布、伸缩和管理。  
Kubernetes 是谷歌开源的容器集群管理系统，是 Google 多年大规模容器管理技术 Borg 的开源版本  
主要特点是:

- 自动上线和回滚
- 服务发现和负载均衡
- Service 拓扑
- 存储业务流程->自动挂载存储系统
- 安全和便捷的配置管理
- 自动部署容器
- 支持批量执行
- IPv4/IPv6双协议栈
- 横向扩展
- 自我修复

## 2. 核心组件

Kubernetes重要由以下几个核心组件组成:

- apiserver 提供了资源操作的唯一入口，并提供认证、授权、访问控制、API 注册和发现等机制
- etcd 保存整个集群的状态，是集群数据的后台存储区
- scheduler 负责资源的调度，按照预定的调度策略将 Pod 调度到相应的机器上
- controller-manager 负责维护集群的状态，比如发现和响应节点下线、故障检测、自动扩展、滚动更新等
- cloud-controller-manager 云端控制管理中心，依赖于云供应商
- kubelet 集群中每个节点的代理，维护容器的生命周期，同时也负责 Volume（CVI）和网络（CNI）的管理
- kube-proxy 在节点中维护网络规则，维持集群内外的通信
- container-runtime 在节点中负责容器运行管理和镜像管理

### 2.1. API server

API server是Kubernetes控制端的核心组件, API server开放一个HTTP API, 以允许用户-集群-外部组件三者彼此通信  
允许用户查询和管理Kubernetes API中的对象状态(比如: Pods, Namespace, ConfigMaps, Events)  
多数操作可以通过kubectl和kubeadm命令来完成

## 3. Kuberbetes对象
