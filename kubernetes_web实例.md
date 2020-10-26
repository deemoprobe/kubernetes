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
kind: ReplicationController
metadata:
  name: mysql
spec:
  replicas: 1
  selector:
    app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql
        ports:
        - containerPort: 3306
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
        - containerPort: 8090
        env:
        - name: MYSQL_SERVICE_HOST
          value: 192.168.68.128


kubectl create -f myweb_rc.yaml

# 创建 SVC
vi myweb_svc.yaml

apiVersion: v1
kind: Service
metadata:
  name: myweb
spec:
  type: NodePort
  ports:
    - port: 8090
      nodePort: 30001
  selector:
    app: myweb

kubectl create -f myweb_svc.yaml
```
