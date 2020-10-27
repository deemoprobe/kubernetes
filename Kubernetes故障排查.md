# Kubernetes service中的故障排查

- 查看某个资源的定义和用法

```bash
kubectl explain
```

- 查看Pod的状态

```bash
kubectl get pods
kubectl describe pods my-pod
```

- 监控Pod状态的变化

```bash
kubectl get pod -w
```

可以看到一个 namespace 中所有的 pod 的 phase 变化。

- 查看 Pod 的日志

```bash
kubectl logs my-pod
kubectl logs my-pod -c my-container
kubectl logs -f my-pod
kubectl logs -f my-pod -c my-container
```

`-f` 参数可以 follow 日志输出。

- 交互式 debug

```bash
kubectl exec my-pod -it /bin/bash
kubectl top pod POD_NAME --containers
```

- 查看访问地址

    Endpoint = Pod IP + Container Port

```shell
[root@k8s-master ~]# kubectl get endpoints
NAME         ENDPOINTS                               AGE
kubernetes   192.168.43.236:6443                     28h
mysql        192.168.36.85:3306                      24h
myweb        192.168.36.84:8080,192.168.36.86:8080   24h
nginx        192.168.36.87:80                        28h
```

- 查看Service详情信息

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

- 查看namespace

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
