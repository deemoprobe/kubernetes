# Kubernetes 网络实现过程

Kubernetes网路包括网络模型、CNI、Service、Ingress、DNS等。

在Kubernetes网络模型中，每台服务器都有自己的IP段，各个服务器之间的容器可以根据目标服务器IP进行地址访问。

![20201127132701](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20201127132701.png)

要实现这种效果需要解决各个服务器之间的**IP地址分配**和访问**路由的建立**。

## Kubernetes网络
