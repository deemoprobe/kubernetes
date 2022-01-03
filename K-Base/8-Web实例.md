# Kubernetes基础之Web实例

## 实例说明

创建运行在Tomcat里面的Web, 实现JSP页面通过jdbc直接访问MySQL数据库在页面上展示数据.  
需要两个容器: MySQL 和 Tomcat-Web

## 创建MySQL

```shell
vi mysql.yaml

apiVersion: apps/v1
# 定义为deployment
kind: Deployment
metadata:
  # deploy的名称,全局唯一
  name: mysql
  labels:
    app: mysql
spec:
  # 希望创建的pod副本数
  replicas: 1
  selector:
    # 选择符合该标签的pod
    matchLabels:
      app: mysql
  # 根据模板下的定义来创建pod
  template:
    metadata:
      labels:
        # pod的标签,对应selector
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

kubectl create -f mysql.yaml

# 创建 SVC
vi mysqlsvc.yaml

apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
    - port: 3306
  selector:
    app: mysql

kubectl create -f mysqlsvc.yaml

[root@k8main manifests]# kubectl get pods,svc
NAME                         READY   STATUS    RESTARTS   AGE
pod/mysql-8d27z              1/1     Running   0          6m42s

NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
service/mysql        ClusterIP   192.168.68.128   <none>        3306/TCP       10s

# 查看pod状态
kubectl describe po mysql
```

## 创建 Web

```shell
vi web.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: kubeguide/tomcat-app:v1
        ports:
        - containerPort: 8080
        env:
        - name: MYSQL_SERVICE_HOST
          # 这里的IP是mysql_svc的虚拟IP(CLUSTER-IP)
          value: "192.168.68.128"
        - name: MYSQL_SERVICE_PORT
          value: "3306"

kubectl create -f web.yaml

# 创建 SVC
vi websvc.yaml

apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  type: NodePort
  ports:
    # 本地服务的8080端口映射到node节点端口30001
  - port: 8080
    nodePort: 30001

kubectl create -f myweb_svc.yaml
```

## 访问结果

访问地址：IP:30001/demo/

![20201026143020](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20201026143020.png)
