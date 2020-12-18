# Kubernetes存储之Download API

引用Downward API元数据信息的常用方式之一是使用容器的环境变量、它通过在valueFrom字段中嵌套fieldRef或resourceFieldRef字段来引用相应的数据源, 也可以为存储卷注入元数据

Download API作用: 可以通过环境变量或Volume挂载将pod信息注入到容器内部

fieldRef字段引用的信息具体如下：

```shell
pod.spec.nodeName              #node节点名称
pod.spec.serviceAccountName    #pod对象使用的serviceAccount资源名称
pod.status.hostIP              #pod的IP地址
pod.status.podIP               #pod对象的IP地址
pod.metadata.name              #pod对象的名称
pod.metadata.namespace         #pod对象隶属的名称空间
pod.metadata.labels            #pod对象标签中的指定键的值
pod.metadata.annotations       #pod对象注解信息中的指定键的值
```

## 1. 实例

### 1.1. 注入环境变量

```shell
[root@k8s-master volume]# mkdir dlapi;cd dlapi
[root@k8s-master dlapi]# vi dlapi_demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: dlapi-demo
spec:
  containers:
  - name: dlapi-demo
    image: busybox
    command: [ "/bin/sh", "-c", "env" ]
    env:
    - name: MY_POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: MY_POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: MY_POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
  restartPolicy: Never
# 创建并查看
[root@k8s-master dlapi]# kubectl apply -f dlapi_demo.yaml 
pod/dlapi-demo created
[root@k8s-master dlapi]# kubectl get po
NAME                            READY   STATUS              RESTARTS   AGE
dlapi-demo                      0/1     Completed           0          47s
# 查看日志, 可见三个注入的环境变量已经打印出来了
[root@k8s-master dlapi]# kubectl logs dlapi-demo | grep MY
MY_POD_NAMESPACE=default
MY_POD_IP=172.16.36.111
MY_POD_NAME=dlapi-demo
```

### 1.2. 存储卷注入元数据

```shell
# 创建yaml
[root@k8s-master dlapi]# vi dlapi_demo_vol.yaml
kind: Pod
apiVersion: v1
metadata:
  labels:
    zone: east-china
    cluster: downward-api-test-cluster1
    rack: rack-101
    app: dapi-vol-pod
  name: dapi-vol-pod
kind: Pod
apiVersion: v1
metadata:
  labels:
    zone: Shanghai-China
    cluster: downward-api-test-cluster1
    app: dlapi-demo-vol
  name: dlapi-demo-vol
  annotations:
    annotation1: "test-value-1"
    annotation2: "test-value-2"
spec:
  containers:
    - name: volume-test-container
      image: busybox
      command: ["sh", "-c", "sleep 864000"]
      resources:
        requests:
          memory: "32Mi"
          cpu: "125m"
        limits:
          memory: "64Mi"
          cpu: "250m"
      volumeMounts:
      - name: podinfo
        mountPath: /etc/podinfo
        readOnly: false
  volumes:
  - name: podinfo
    downwardAPI:
      defaultMode: 420
      items:
      - fieldRef:
          fieldPath: metadata.name
        path: pod_name
      - fieldRef:
          fieldPath: metadata.namespace
        path: pod_namespace
      - fieldRef:
          fieldPath: metadata.labels
        path: pod_labels
      - fieldRef:
          fieldPath: metadata.annotations
        path: pod_annotations
      - resourceFieldRef:
          containerName: volume-test-container
          resource: limits.cpu
        path: "cpu_limit"
      - resourceFieldRef:
          containerName: volume-test-container
          resource: requests.memory
          divisor: "1Mi"
        path: "mem_request"
[root@k8s-master dlapi]# kubectl get po dlapi-demo-vol
NAME             READY   STATUS    RESTARTS   AGE
dlapi-demo-vol   1/1     Running   0          20s
# 查看pod_labels
[root@k8s-master dlapi]# kubectl exec dlapi-demo-vol -- cat /etc/podinfo/pod_labels
app="dlapi-demo-vol"
cluster="downward-api-test-cluster1"
zone="Shanghai-China"
# 查看pod_name
[root@k8s-master dlapi]# kubectl exec dlapi-demo-vol -- cat /etc/podinfo/pod_name
dlapi-demo-vol
# 查看pod_namespace
[root@k8s-master dlapi]# kubectl exec dlapi-demo-vol -- cat /etc/podinfo/pod_namespace
default
# 增加新label
[root@k8s-master dlapi]# kubectl label po dlapi-demo-vol master="192.168.3.43"
pod/dlapi-demo-vol labeled
# 查看pod_labels
[root@k8s-master dlapi]# kubectl exec dlapi-demo-vol -- cat /etc/podinfo/pod_labels
app="dlapi-demo-vol"
cluster="downward-api-test-cluster1"
master="192.168.3.43"
zone="Shanghai-China"
```
