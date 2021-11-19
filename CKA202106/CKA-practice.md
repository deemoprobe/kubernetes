# CKA模拟练习

## 注意事项

- 考试总时长为2小时，共17道题，有一次补考机会，考试结束后36小时内通过电子邮件发送结果
- 在线考试，只使用chrome浏览器，不会使用到其他软件
- 硬件要求麦克风、摄像头，摄像头可以移动，考前至少提前15分钟进入网站，配合考官对考试环境做检查
- 考试要求房间安静不允许外人进出，咖啡厅以及开放式办公环境不可以。桌面及桌子底下不允许摆放杂物，纸、电子产品、垃圾桶等一律不允许
- 网络需要稳定，国外的服务器，国内访问比较慢，建议搭梯子访问<https://www.examslocal.com/ScheduleExam/Home/CompatibilityCheck> 检测硬件及网络
- 建议使用护照来作为主身份信息，身份证需要配合其他带签名的证件一起才可以
- 除考试窗口外，只允许使用Chrome浏览器额外打开一个窗口，
<https://kubernetes.io/docs/>
<https://github.com/kubernetes/>
<https://kubernetes.io/blog/> 及其子域。这包括这些页面的所有可用语言翻译（例如<https://kubernetes.io/zh/docs/>）
- 复制和粘贴，Ctrl + Insert复制并Shift + Insert粘贴
- 考试环境为Kubernetes 1.22版本，主机操作系统为Ubuntu，共有6套环境，通过 `kubectl config use-context xxxx` 进行切换，每道题目开始会提示如何切换环境，默认是在student@node-1终端，可以通过ssh 来访问集群里的其他节点，通过sudo -i 切换到root用户

## 1.RBAC

```shell
# Context
You have been asked to create a new ClusterRole for a deployment pipeline and bind it to a specific ServiceAccount scoped to a specific namespace.
# Task
Create a new Cluster Role named deployment-clusterrole,which only allows to create the following resource types:
- Deployment
- StatefuleSet
- DaemonSet
Create a new ServiceAccount named cicd-token in the existing namespace app-team1.Bind the new ClusterRole deployment-clusterrole to the new ServiceAccount cicd-token, limited to the namespace app-team1.
```

```shell
kubectl config use-context k8s
```

## 2.cordon & drain

```shell
# Task
Set the node named ek8s-node-1 as unavailable and reschedule all the pods running on it.
```

```shell
kubectl config use-context ek8s

```

## 3.update cluster

```shell
# Task
Given an existing Kubernetes cluster running version 1.21.0, upgrade all the Kubernetes control plane and node components on the master node only to version 1.22.0
```

```shell
kubeclt config use-context mk8s
```

## 4.ETCD backup

```shell
# Task
First, create a snapshot of the existing etcd instance running at https://127.0.0.1:2379, saving the snapshot to /srv/data/etcd-snapshot.db.Creating a snapshot of the given instance is expected to complete in seconds.If the operation seems to hang, something is likely wrong with your command. Use ctrl+c to cancel the operation and try again.

Next, restore an existing, previous snapshot located at /var/lib/backup/etcd-snapshot-previous.db. The following TLS certificates/key are supplied for connecting to the server with etcdctl:
- CA certificate: /opt/KUIN00601/ca.crt
- Client certificate: /opt/KUIN00601/etcd-client.crt
- Client key: /opt/KUIN00601/etcd-client.key
```

```shell

```

## 5.NetworkPolicy

```shell
# Task
Create a new NetworkPolicy name allow-port-from-namespace that allows Pods in the existing namespace internal to connect to port 9000 of other Pods in the same namespace.

Ensure that the new NetworkPolicy:
- does not allow access to Pods not listening on port 9000
- does not allow access from Pods not in namespace internal
```

```shell
kubectl config use-context hk8s
```

## 6.Service

```shell
# Task
Reconfigure the existing deployment front-end and add a port specification name http exposing port 80/tcp of the existing container nginx.

Creat a new service named front-end-svc exposing the container port http.

Configure the new service to also expose the individual Pods via a NodePort on the node on which they are scheduled.
```

```shell
kubectl config use-context k8s
```

## 7.Ingress

```shell
# Task
Create a new nginx Ingress resource as follows:
- Name: pong
- Namespace: ing-internal
- Exposing service hi on path /hi using service port 5678

The availability of service hi can be checked using the following command, which shoud return hi:
[student@node-1] $ curl -kL <INTERNAL_IP>/hi
```

```shell
kubectl config use-context k8s
```

## 8.Replicas Deployment

```shell
# Task
Scale the deployment presentation to 3 pods.
```

```shell
kubectl config use-context k8s
```

## 9.Schedule Pod

```shell
# Task
Schedule a pod as follows:
- Name: nginx-kusc00401
- Image: nginx
- Node selector: disk=spinning
```

```shell
kubectl config use-context k8s
```

## 10.Check node

```shell
# Task
Check to see how many nodes are ready(not including nodes tainted NoSchedule) and write the number to /opt/KUSC00402/kusc00402.txt
```

```shell
kubectl config use-context k8s
```

## 11.Images Pod

```shell
Create a pod named kucc8 with a single app container for each of the following images rinning inside(there may be between 1 and 4 images specified):
nginx + redis + memcached + consul
```

```shell
kubectl config use-context k8s
```

## 12.PV

```shell
# Task
Create a persistent volume with name app-config, of capacity 1Gi and access mode ReadOnlyMany. The type of volume is hostPath and its location is /srv/app-config
```

```shell
kubectl config use-context hk8s
```

## 13.PVC

```shell
# Task
Create a new PersistentVloumeClaim:
- Name: pv-volume
- Class: csi-hostpath-sc
- Capacity: 10Mi

Create a new Pod which mounts the PersistentVolumeClaim as a volume:
- Name: web-server
- Image: nginx
- Mount path: /usr/share/nginx/html

Configure the new pod to have ReadWriteOnce access to the volume

Finally, using kubectl edit or kubectl patch expand the PersistentVolumeClaim to a capacity of 70Mi and record the change
```

```shell
kubectl config use-context ok8s
```

## 14.Save Pod Error Log

```shell
# Task
Monitor the logs of pod bar and:
- Extract log lines corresponding to error unable-to-access-website
- Write them to /opt/KUTR00101/bar
```

```shell
kubectl config use-context k8s
```

## 15.SideCar

```shell
# Context
Without changing its existing containers, anexisting Pod needs to be integrated into Kubernetes built-in logging architecure(e.g. kubectl logs). Adding a streaming sidecar conatainer is a good and common way to accomplish this requirement.
# Task
Add a busybox sidecar container to the existing Pod big-corp-app. The new sidecar container has to run the following command: /bin/sh -c tail -n+1 /var/log/big-corp-app.log
Use a volume mount named logs to make the file /var/log/big-corp-app.log available to sidecar container.

Do not modify the existing container. Do not modify the path of the log file, both containers must access it at /var/log/big-corp-app.log
```

```shell

```

## 16.Top

```shell
# Task
From the pod label name=cpu-loader, find pods running high CPU wordloads and write the name of the pod consuming most CPU to the file /opt/KUTR00401/KUTR00401.txt(which already exists)
```

```shell
kubectl config use-context k8s
```

## 17.Node NotReady Check

```shell
# Task
A Kubernetes work node named wk8s-node-0 is in state NotReady. Investigate why this is the case, and perform any appropriate step to bring the node to a Ready state., ensuring that any changes are made permanent.

You can ssh to the failed node using: ssh wk8s-node-0
You can assume elevated privileges on the node with the following command: sudo -i
```

```shell
kubectl config use-context wk8s
```
