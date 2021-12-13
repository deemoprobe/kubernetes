# ETCD备注

## 读请求处理流程

![etcd读请求处理流程](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/etcd读请求处理流程.png)

## 写请求处理流程

![etcd写请求处理流程](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/etcd写请求处理流程.png)

## v2和v3版本的差异

v2版本的缺陷如图：

![1639352574704](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/1639352574704.png)

v3版本采用了gPRC API

gRPC是基于HTTP/2.x的一种现代开源高性能远程过程调用 (RPC=Remote Procedure Call) 框架，可以在任何环境中运行。它可以通过对负载平衡、跟踪、健康检查和身份验证的可插拔支持，有效地连接数据中心内和数据中心之间的服务。它还适用于分布式计算的最后一英里，将设备、移动应用程序和浏览器连接到后端服务。
