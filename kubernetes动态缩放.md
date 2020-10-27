# kubernetes动态缩放

- 当前RC和pod情况是

```shell
[root@k8s-master ~]# kubectl get pods,rc
NAME                         READY   STATUS    RESTARTS   AGE
pod/mysql-hdg66              1/1     Running   3          24h
pod/myweb-ctzhn              1/1     Running   3          24h
pod/myweb-dm94j              1/1     Running   3          24h
pod/nginx-6799fc88d8-xj4c4   1/1     Running   4          28h

NAME                          DESIRED   CURRENT   READY   AGE
replicationcontroller/mysql   1         1         1       24h
replicationcontroller/myweb   2         2         2       24h
```

- 增加pod-mysql的副本(RC)数

```shell
[root@k8s-master ~]# kubectl scale rc mysql --replicas=3
replicationcontroller/mysql scaled
[root@k8s-master ~]# kubectl get pods,rc
NAME                         READY   STATUS    RESTARTS   AGE
pod/mysql-bqdvv              1/1     Running   0          5s
pod/mysql-hdg66              1/1     Running   3          24h
pod/mysql-hhb6t              1/1     Running   0          5s
pod/myweb-ctzhn              1/1     Running   3          24h
pod/myweb-dm94j              1/1     Running   3          24h
pod/nginx-6799fc88d8-xj4c4   1/1     Running   4          28h

NAME                          DESIRED   CURRENT   READY   AGE
replicationcontroller/mysql   3         3         3       24h
replicationcontroller/myweb   2         2         2       24h
```

- 减少pod-mysql的副本(RC)数

```shell
[root@k8s-master ~]# kubectl scale rc mysql --replicas=1
replicationcontroller/mysql scaled
# 正在停止多余的副本
[root@k8s-master ~]# kubectl get pods,rc
NAME                         READY   STATUS        RESTARTS   AGE
pod/mysql-bqdvv              1/1     Terminating   0          62s
pod/mysql-hdg66              1/1     Running       3          24h
pod/mysql-hhb6t              1/1     Terminating   0          62s
pod/myweb-ctzhn              1/1     Running       3          24h
pod/myweb-dm94j              1/1     Running       3          24h
pod/nginx-6799fc88d8-xj4c4   1/1     Running       4          28h

NAME                          DESIRED   CURRENT   READY   AGE
replicationcontroller/mysql   1         1         1       24h
replicationcontroller/myweb   2         2         2       24h
[root@k8s-master ~]# kubectl get pods,rc
NAME                         READY   STATUS    RESTARTS   AGE
pod/mysql-hdg66              1/1     Running   3          24h
pod/myweb-ctzhn              1/1     Running   3          24h
pod/myweb-dm94j              1/1     Running   3          24h
pod/nginx-6799fc88d8-xj4c4   1/1     Running   4          28h

NAME                          DESIRED   CURRENT   READY   AGE
replicationcontroller/mysql   1         1         1       24h
replicationcontroller/myweb   2         2         2       24h
```
