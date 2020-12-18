# Kubernetes K8S网络之Service

简单来说, Kubernetes_Service代表的是Kubernetes后端服务的入口, 包含服务的访问IP(Cluster IP)和端口[Kubernetes的Endpoints Controller会生成一个Endpoints对象, 记录 Endpoints=ClusterIP+Port], 通过Label Selector选择与之匹配的Pod.

## 实例
