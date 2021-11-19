# Kubernetes基础之相关概念

## Kubernetes是什么

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

## 核心组件

Kubernetes重要由以下几个核心组件组成:

- apiserver 提供了资源操作的唯一入口，并提供认证、授权、访问控制、API 注册和发现等机制
- controller-manager 负责维护集群的状态，比如发现和响应节点下线、故障检测、自动扩展、滚动更新等
- scheduler 负责资源的调度，按照预定的调度策略将 Pod 调度到相应的机器上
- kubelet 集群中每个节点的代理，维护容器的生命周期，同时也负责 Volume（CVI）和网络（CNI）的管理
- kube-proxy 在节点中维护网络规则，维持集群内外的通信
- container-runtime 在节点中负责容器运行管理和镜像管理
- etcd 保存整个集群的状态，是集群数据的后台存储区，键值数据库
- cloud-controller-manager 云端控制管理中心，依赖于云供应商

## Master节点

整个集群的控制中枢，核心组件如下：

- apiserver 集群的控制中心，各个模块之间信息交互都需要经过apiserver，同时也是集群管理、资源配额、安全机制的入口
- controller-manager 集群的状态管理器，保证pod或其他资源达到期望值，也是需要和apiserver进行通信，在需要的时候可以创建、更新、删除相应的资源
- scheduler 集群资源调度器，根据一系列预定的条件对资源进行调度
- etcd 键值数据库

master节点资源一定要尽量给够，以至于后期不会拖累集群的整体性能。并且允许的情况下，etcd最好也要和master节点区分开来，单独创建集群进行数据的存储，etcd-cluster必须使用高性能ssd硬盘，否则后期将大大影响集群的性能

## Node节点

应用部署的节点，工作节点，资源调度的对象，核心组件如下：

- kubelet 负责监听节点上pod的状态，同时负责将状态上报给master节点，与master节点进行通信
- kube-proxy 负责pod之间的通信和负载均衡，将指定的流量分发到后端正确的机器上，一般工作模式默认为ipvs

IPVS：监听master节点增加和删除service以及endpoint的消息，调用netlink接口创建相应的ipvs规则。通过ipvs规则将流量转到相应的pod上。ipvs是内核级的转发，速度很快  
Iptables：监听master节点增加和删除service以及endpoint的消息，对于每一个service，都会创建一个iptables规则，将service的clusterIP代理到后端对应的pod上。当规则很多时，性能会比ipvs差，所以一般选择ipvs即可。

## Calico

符合CNI标准的网络插件，会给每个pod生成唯一的IP地址，并且把每个节点当做一个路由器。可以使用`route -n`看一下。

## CoreDNS

用于集群内部service的解析，可以让pod把service名称解析成ip地址，然后通过service的IP地址链接到对应的应用上。

## Pod

Pod是Kubernetes中最小的单元，是由一个或多个容器组成的。每个pod还包含一个pause容器，pause容器是pod的父容器，负责僵尸进程的回收管理，通过pause容器可以使同一个pod内多个容器共享存储、网络、PID、IPC等。
