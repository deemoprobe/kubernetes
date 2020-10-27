# kubernetes web 实例

## 实例说明

创建运行在Tomcat里面的Web APP, 实现JSP页面通过jdbc直接访问MySQL数据库在页面上展示数据.  
需要两个容器: Web APP 和 MySQL

## 创建MySQL

```shell
cd /etc/kubernetes/manifests
# 创建RC
vi mysql_rc.yaml

apiVersion: v1
# 定义为 RC (副本控制器)
# ReplicationSet目前在替代ReplicationController的写法,意义相同
kind: ReplicationController
metadata:
  # RC的名称,全局唯一
  name: mysql
spec:
  # 希望创建的pod个数
  replicas: 1
  selector:
    # 选择符合该标签的pod
    app: mysql
  # 根据模板下的定义来创建pod
  template:
    metadata:
      labels:
        # pod的标签,对应RC的selector
        app: mysql
    # 定义pod规则
    spec:
      # pod内容器的定义
      containers:
      # 容器名称
      - name: mysql
        # 容器所使用的的镜像(不指定版本的话就默认拉取最新版)
        # 由于最新版驱动的问题, 所以最好使用指定版本
        image: mysql:5.6
        ports:
        # 开放的端口号
        - containerPort: 3306
        # 容器环境变量
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "123456"

kubectl create -f mysql_rc.yaml

# 创建 SVC
vi mysql_svc.yaml

apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
    - port: 3306
  selector:
    app: mysql

kubectl create -f mysql_svc.yaml

[root@k8main manifests]# kubectl get pods,svc
NAME                         READY   STATUS    RESTARTS   AGE
pod/mysql-8d27z              1/1     Running   0          6m42s

NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
service/mysql        ClusterIP   192.168.68.128   <none>        3306/TCP       10s

# 查看pod状态
kubectl describe po mysql
```

## 创建 MyWeb APP

```shell
# 创建RC
vi myweb_rc.yaml

apiVersion: v1
kind: ReplicationController
metadata:
  name: myweb
spec:
  replicas: 2
  selector:
    app: myweb
  template:
    metadata:
      labels:
        app: myweb
    spec:
      containers:
      - name: myweb
        image: kubeguide/tomcat-app:v1
        ports:
        - containerPort: 8080
        env:
        - name: MYSQL_SERVICE_HOST
          # 这里的IP是名为MySQL的pod虚拟IP(CLUSTER-IP)
          value: 192.168.68.128
        - name: MYSQL_SERVICE_PORT
          value: "3306"

kubectl create -f myweb_rc.yaml

# 创建 SVC
vi myweb_svc.yaml

apiVersion: v1
kind: Service
metadata:
  name: myweb
spec:
  selector:
    app: myweb
  type: NodePort
  ports:
    # 本地服务的8080端口映射到外部端口30001
    - port: 8080
      nodePort: 30001

kubectl create -f myweb_svc.yaml
```

## 访问结果

![20201026143020](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20201026143020.png)

## 问题总结

1. MySQL版本需要选择5.6
2. 端口访问不通

```shell
# 先打开防火墙, 开放端口再关闭
systemctl start firewalld
firewall-cmd --zone=public --add-port=30001/tcp --permanent
firewall-cmd --reload
systemctl stop firewalld
```
