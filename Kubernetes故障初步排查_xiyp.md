# k8s 近期故障排查总结

### 1.一般故障排查

排查方案:

流程1: kubectl get nodes查看各节点状态-->假设k8s-node1 NotReady-->  
kubectl describe nodes k8s-node1查看节点详细状态-->着重分析Events和Conditions下面输出的提示信息-->  
kubectl describe pods mysql-hdg66查看对应pod的状态-->主要查看Events输出的日志-->  
kubectl describe svc mysql查看对应service的状态-->主要查看Events输出的日志

流程2:  
master节点中查看集群各组件的状态(如果是组件问题,重新拉取镜像即可)  
kubectl get pods -n kube-system

流程3:使用下面命令在对应NotReady节点上获取k8s运行日志
journalctl -f -u kubelet

### 2.`kubectl get cs`查看组件为unhealthy状态

排查方案：

流程：

- kubectl get pods -A查看所有pod是否正常
- 检查kube-scheduler和kube-controller-manager组件配置是否禁用了非安全端口
  vim /etc/kubernetes/manifests/kube-scheduler.yaml  
  vim /etc/kubernetes/manifests/kube-controller-manager.yaml  
  删除`--port=0`这一行  
- 重启kubelet
  systemctl restart kubelet

### 3.查看Pod的状态

```bash
kubectl get pods
kubectl describe pods my-pod
```

### 4.查看 Pod 的日志

```bash
kubectl logs my-pod
kubectl logs my-pod -c my-container
kubectl logs -f my-pod
kubectl logs -f my-pod -c my-container
```

### 5.查看访问地址

Endpoint = Pod IP + Container Port

```shell
[root@k8s-master ~]# kubectl get endpoints
NAME         ENDPOINTS                               AGE
kubernetes   192.168.43.236:6443                     28h
mysql        192.168.36.85:3306                      24h
myweb        192.168.36.84:8080,192.168.36.86:8080   24h
nginx        192.168.36.87:80                        28h
```

### 6.以yaml形式输出配置信息

```shell
[root@k8s-master ~]# kubectl get svc myweb -o yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: "2020-10-26T06:14:53Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:spec:
        f:externalTrafficPolicy: {}
        f:ports:
          .: {}
          k:{"port":8080,"protocol":"TCP"}:
            .: {}
            f:nodePort: {}
            f:port: {}
            f:protocol: {}
            f:targetPort: {}
        f:selector:
          .: {}
          f:app: {}
        f:sessionAffinity: {}
        f:type: {}
    manager: kubectl-create
    operation: Update
    time: "2020-10-26T06:14:53Z"
  name: myweb
  namespace: default
  resourceVersion: "24576"
  selfLink: /api/v1/namespaces/default/services/myweb
  uid: ac76191d-78e2-4786-a930-793a338753ea
spec:
  clusterIP: 192.168.115.169
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 30001
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: myweb
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
```

### 7.查看namespace

```shell
# namespace具有资源隔离的作用
# 查看namespace
[root@k8s-master ~]# kubectl get namespaces
NAME              STATUS   AGE
default           Active   29h
kube-node-lease   Active   29h
kube-public       Active   29h
kube-system       Active   29h
# 默认查看namespace=default内的资源
[root@k8s-master ~]# kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
mysql-hdg66              1/1     Running   3          25h
myweb-ctzhn              1/1     Running   3          25h
myweb-dm94j              1/1     Running   3          25h
nginx-6799fc88d8-xj4c4   1/1     Running   4          29h
# 指定namespace
[root@k8s-master ~]# kubectl get pods --namespace=kube-public
```
