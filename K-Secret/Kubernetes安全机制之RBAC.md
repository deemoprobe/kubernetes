# Kubernetes安全机制之RBAC

Kubernetes通过一系列安全机制来确保集群的安全,包括API Server 的认证授权(Authentication/Authorization)/准入控制机制(Admission Control)/敏感信息的保护机制(Secret/Service_Account)等.

## 认证「Authentication」

- **HTTP Token认证:** 通过一个Token来识别合法用户.HTTP Token的认证是用一个很长的特殊编码方式的并且难以被模仿的字符串来表达客户的一种方式.每一个Token对应一个用户名,存储在API Server能访问的文件中.当客户端发起API调用请求时,需要在HTTP Header里放入Token
- **HTTP Base认证:** 通过用户名+密码的方式认证.用户名:密码 用base64算法进行编码后的字符串放在HTTP Request中的Heather Authorization 域里发送给服务端,服务端收到后进行解码,获取用户名和密码
- **最严格的HTTPS证书认证:** 基于CA根证书签名的客户端身份认证方式

## 授权「Authorization」

认证只是确认通信的双方都是可信的,可以相互通信.而授权是确定请求方有哪些资源的权限.API Server目前支持如下几种授权策略（通过API Server的启动参数 --authorization-mode 设置）

- AlwaysDeny：表示拒绝所有请求.仅用于测试
- AlwaysAllow：表示允许所有请求.如果有集群不需要授权流程,则可以采用该策略
- Node：节点授权是一种特殊用途的授权模式,专门授权由 kubelet 发出的 API 请求
- Webhook：是一种 HTTP 回调模式,允许使用远程 REST 端点管理授权
- ABAC(Attribute-Based Access Control)：基于属性的访问控制,表示使用用户配置的授权规则对用户请求进行匹配和控制
- RBAC（Role-Based Access Control）：基于角色的访问控制,默认使用该规则

## RBAC授权模式

RBAC基于角色的访问控制,在Kubernetes 1.5 中引入,现为默认授权标准.相对其他访问控制方式,拥有如下优势：

- 对集群中的资源和非资源均拥有完整的覆盖
- 整个RBAC完全由几个API对象完成,同其他API对象一样,可以用kubectl或API进行操作
- 可以在运行时进行操作,无需重启API Server

## RBAC API资源对象
