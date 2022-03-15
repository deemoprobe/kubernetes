# Kubernetes配置案例

## 1.RBAC

```shell
# Context
You have been asked to create a new ClusterRole for a deployment pipeline and bind it to a specific ServiceAccount scoped to a specific namespace.
# Task
Create a new ClusterRole named deployment-clusterrole,which only allows to create the following resource types:
- Deployment
- StatefulSet
- DaemonSet
Create a new ServiceAccount named cicd-token in the existing namespace app-team1.Bind the new ClusterRole deployment-clusterrole to the new ServiceAccount cicd-token, limited to the namespace app-team1.
```

[解题参考:RBAC鉴权](https://kubernetes.io/zh/docs/reference/access-authn-authz/rbac/)

```shell
******* Prepare *******
~ kubectl config use-context k8s
```

```shell
******* Answer *******
~ kubectl create clusterrole deployment-clusterrole --verb=create --resource=deployments,statefulsets,daemonsets
~ kubectl -n app-team1 create serviceaccount cicd-token
~ kubectl -n app-team1 create rolebinding deployment-rolebinding --clusterrole=deployment-clusterrole --serviceaccount=app-team1:cicd-token
```

```shell
# 如果没有限定namespace,则需要使用clusterrolebinding
~ kubectl create clusterrolebinding deployment-clusterrolebinding --clusterrole=deployment-clusterrole --serviceaccount=app-team1:cicd-token
# clusterrole和clusterrolebinding不受namespace限制，不需要加 -n namespace
```

## 2.cordon & drain

```shell
# Task
Set the node named ek8s-node-1 as unavailable and reschedule all the pods running on it.
```

[解题参考:Cordon&Drain](https://kubernetes.io/zh/docs/reference/kubectl/cheatsheet/#%E4%B8%8E%E8%8A%82%E7%82%B9%E5%92%8C%E9%9B%86%E7%BE%A4%E8%BF%9B%E8%A1%8C%E4%BA%A4%E4%BA%92)

```shell
******* Prepare *******
~ kubectl config use-context ek8s
******* Answer *******
~ kubectl cordon ek8s-node-1
~ kubectl drain ek8s-node-1 --ignore-daemonsets --delete-emptydir-data --force
```

## 3.Cluster upgrade

```shell
# Task
Given an existing Kubernetes cluster running version 1.22.3, upgrade all the Kubernetes control plane and node components on the master node only to version 1.22.4
You can also expected to upgrade kubelet and kubectl on the master node.
Be sure drain the master node before upgrading it and uncordon it after upgrading. Do not upgrade the worker nodes, etcd,the container manager, the CNI plugin, the DNS server and any other addons.
```

[解题参考:升级集群](https://kubernetes.io/zh/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)

```shell
******* Prepare *******
~ kubectl config use-context mk8s
******* Answer *******
# 查看
~ kubectl get nodes
~ kubectl drain mk8s-master-1 --ignore-daemonsets --delete-emptydir-data --force
# 切换到要升级的对应版本的master节点上
~ ssh mk8s-master-1
~ sudo -i
~ apt update
~ apt-cache madison kubeadm | grep 1.22.4
~ apt install kubeadm=1.22.4-00 -y
~ kubeadm version
~ kubeadm upgrade plan
# 重点是要不要忽略etcd，注意审题，忽略时参数--etcd-upgrade=false
~ kubeadm upgrade apply 1.22.4 --etcd-upgrade=false
~ apt-get install kubelet=1.22.4-00 -y
~ apt-get install kubectl=1.22.4-00 -y
~ kubelet --version
~ kubectl version
~ exit
# 退出节点后执行
~ kubectl uncordon mk8s-master-1
```

## 4.ETCD backup and restore

```shell
# Task
First, create a snapshot of the existing etcd instance running at https://127.0.0.1:2379, saving the snapshot to /srv/data/etcd-snapshot.db.Creating a snapshot of the given instance is expected to complete in seconds.If the operation seems to hang, something is likely wrong with your command. Use ctrl+c to cancel the operation and try again.

Next, restore an existing, previous snapshot located at /var/lib/backup/etcd-snapshot-previous.db. The following TLS certificates/key are supplied for connecting to the server with etcdctl:
- CA certificate: /opt/KUIN00601/ca.crt
- Client certificate: /opt/KUIN00601/etcd-client.crt
- Client key: /opt/KUIN00601/etcd-client.key
```

[解题参考:ETCD备份与恢复](https://kubernetes.io/zh/docs/tasks/administer-cluster/configure-upgrade-etcd/#%E4%BD%BF%E7%94%A8-etcdctl-%E9%80%89%E9%A1%B9%E7%9A%84%E5%BF%AB%E7%85%A7)

```shell
******* Prepare *******
# 如果没有etcdctl命令，安装一下etcd-client
~ apt-get install etcd-client
******* Answer *******
~ ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/opt/KUIN00601/ca.crt --cert=/opt/KUIN00601/etcd-client.crt --key=/opt/KUIN00601/etcd-client.key snapshot save /srv/data/etcd-snapshot.db
~ ETCDCTL_API=3 etcdctl snapshot restore /var/lib/backup/etcd-snapshot-previous.db
```

## 5.NetworkPolicy

```shell
******* Type1 *******
# Task
Create a new NetworkPolicy name allow-port-from-namespace that allows Pods in the existing namespace internal to connect to port 9000 of other Pods in the same namespace.

Ensure that the new NetworkPolicy:
- does not allow access to Pods not listening on port 9000
- does not allow access from Pods not in namespace internal
```

[解题参考:网络策略](https://kubernetes.io/zh/docs/concepts/services-networking/network-policies/#networkpolicy-resource>)

```shell
******* Prepare *******
~ kubectl config use-context hk8s
******* Answer1 *******
~ vim networkpolicy.yaml 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-port-from-namespace
  namespace: internal
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 9000
~ kubectl apply -f networkpolicy.yaml
```

```shell
******* Type2 *******
# Task
Create a new NetworkPolicy named allow-port-from-namespace in the existing namesapce internal that allows Pods in namespace big-corp to connect to port 9000 of the Pods in namespace internal.

Ensure that the new NetworkPolicy:
- does not allow access to Pods not listening on port 9000
- does not allow access from Pods not in namespace big-corp
```

```shell
******* Prepare *******
~ kubectl config use-context hk8s
******* Answer2 *******
~ vim networkpolicy.yaml 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-port-from-namespace
  namespace: internal
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}
    - namespaceSelector:
        matchLabels:
          name: big-corp
    ports:
    - protocol: TCP
      port: 9000
~ kubectl apply -f networkpolicy.yaml
# 查看一下big-corp这个namespace是否有name=big-corp这个标签，如果没有则创建一下
~ kubectl label ns big-corp name=big-corp
```

```shell
******* Type3 *******
# Task
Create a new NetworkPolicy named allow-port-from-namespace in the existing namesapce internal that allows Pods in namespace internal to connect to port 9000 of the Pods in namespace big-corp.

Ensure that the new NetworkPolicy:
- does not allow access to Pods not listening on port 9000
- does not allow access from Pods not in namespace internal
```

```shell
******* Prepare *******
~ kubectl config use-context hk8s
******* Answer3 *******
~ vim networkpolicy.yaml 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-port-from-namespace
  namespace: internal
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 9000
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: big-corp
    ports:
    - protocol: TCP
      port: 9000
~ kubectl apply -f networkpolicy.yaml
# 查看一下big-corp这个namespace是否有name=big-corp这个标签，如果没有则创建一下
~ kubectl label ns big-corp name=big-corp
```

## 6.Service

```shell
# Task
Reconfigure the existing deployment front-end and add a port specification name http exposing port 80/tcp of the existing container nginx.

Creat a new service named front-end-svc exposing the container port http.

Configure the new service to also expose the individual Pods via a NodePort on the node on which they are scheduled.
```

[解题参考:创建Service](https://kubernetes.io/zh/docs/concepts/services-networking/connect-applications-service/)

```shell
******* Prepare *******
~ kubectl config use-context k8s
******* Answer *******
~ kubectl edit deploy front-end
...
      containers:
      - name: nginx
        ports:
        - containerPort: 80
          name: http
          protocol: TCP
...
~ kubectl expose deploy front-end --name=front-end-svc  --port=80 --target-port=http --type=NodePort
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

[解题参考:Ingress](https://kubernetes.io/zh/docs/concepts/services-networking/ingress/#the-ingress-resource)

```shell
******* Prepare *******
~ kubectl config use-context k8s
******* Answer *******
~ vim ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: pong
  namespace: ing-internal
spec:
  rules:
  - http:
      paths:
      - path: /hi
        pathType: Prefix
        backend:
          service:
            name: hi
            port:
              number: 5678
~ kubectl apply -f ingress.yaml
```

## 8.Replicas Deployment

```shell
# Task
Scale the deployment presentation to 3 pods.
```

[解题参考:资源伸缩](https://kubernetes.io/zh/docs/reference/kubectl/cheatsheet/#%E5%AF%B9%E8%B5%84%E6%BA%90%E8%BF%9B%E8%A1%8C%E4%BC%B8%E7%BC%A9)

```shell
******* Prepare *******
~ kubectl config use-context k8s
******* Answer *******
~ kubectl scale deployment presentation --replicas=3
```

## 9.Schedule Pod

```shell
# Task
Schedule a pod as follows:
- Name: nginx-kusc00401
- Image: nginx
- Node selector: disk=spinning
```

[解题参考:调度Pod](https://kubernetes.io/zh/docs/concepts/scheduling-eviction/assign-pod-node/)

```shell
******* Prepare *******
~ kubectl config use-context k8s
******* Answer *******
~ vim nginx-kusc00401.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-kusc00401
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disk: spinning
~ kubectl apply -f nginx-kusc00401.yaml
```

## 10.Check node

```shell
# Task
Check to see how many nodes are ready(not including nodes tainted NoSchedule) and write the number to /opt/KUSC00402/kusc00402.txt
```

```shell
******* Prepare *******
~ kubectl config use-context k8s
******* Answer *******
~ kubectl get nodes | grep -i ready
~ kubectl describe nodes | grep -i taint
~ echo number > /opt/KUSC00402/kusc00402.txt
```

## 11.Images Pod

```shell
Create a pod named kucc8 with a single app container for each of the following images rinning inside(there may be between 1 and 4 images specified):
nginx + redis + memcached + consul
```

[解题参考:创建Pod](https://kubernetes.io/zh/docs/concepts/scheduling-eviction/assign-pod-node/#%E6%AD%A5%E9%AA%A4%E4%BA%8C-%E6%B7%BB%E5%8A%A0-nodeselector-%E5%AD%97%E6%AE%B5%E5%88%B0-pod-%E9%85%8D%E7%BD%AE%E4%B8%AD)

```shell
******* Prepare *******
~ kubectl config use-context k8s
******* Answer *******
~ vim kucc8.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  - name: redis
    image: redis
  - name: memcached
    image: memcached
  - name: consul
    image: consul
~ kubectl apply -f kucc8.yaml
```

## 12.PV

```shell
# Task
Create a persistent volume with name app-config, of capacity 1Gi and access mode ReadOnlyMany. The type of volume is hostPath and its location is /srv/app-config
```

[解题参考:创建PV](https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-persistent-volume-storage/#%E5%88%9B%E5%BB%BA-persistentvolume)

```shell
******* Prepare *******
~ kubectl config use-context hk8s
******* Answer *******
~ vim pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: app-config
  labels:
    type: local
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadOnlyMany
  hostPath:
    path: "/srv/app-config"
~ kubectl apply -f pv.yaml
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

[解题参考:创建PVC](https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-persistent-volume-storage/#%E5%88%9B%E5%BB%BA-persistentvolumeclaim)

```shell
******* Prepare *******
~ kubectl config use-context ok8s
******* Answer *******
~ vim pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pv-volume
spec:
  storageClassName: csi-hostpath-sc
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Mi
~ kubectl apply -f pvc.yaml
~ vim pvc-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-server
spec:
  volumes:
    - name: pv-volume
      persistentVolumeClaim:
        claimName: pv-volume
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: pv-volume
~ kubectl apply -f pvc-pod.yaml
# 编辑大小为70Mi
~ kubectl edit pvc pv-volume --record
```

## 14.Save Pod Error Log

```shell
# Task
Monitor the logs of pod bar and:
- Extract log lines corresponding to error unable-to-access-website
- Write them to /opt/KUTR00101/bar
```

```shell
******* Prepare *******
~ kubectl config use-context k8s
******* Answer *******
~ kubectl logs bar | grep "unable-to-access-website" >> /opt/KUTR00101/bar
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

[解题参考:日志分离Sidecar](https://kubernetes.io/zh/docs/concepts/cluster-administration/logging/#sidecar-container-with-logging-agent)

```shell
******* Prepare *******
~ kubectl config use-context k8s
******* Answer *******
~ kubectl get pod big-corp-app -oyaml > sidecar.yaml
~ vim sidecar.yaml
apiVersion: v1
kind: Pod
metadata:
  name: big-corp-app
spec:
  containers:
  - name: count
    image: busybox
    args:
    - /bin/sh
    - -c
    - >
      i=0;
      mkdir /var/log;
      while true;
      do
        echo "$(date) INFO $i" >> /var/log/big-corp-app.log;
        i=$((i+1));
        sleep 1;
      done
    # 添加下面内容
    volumeMounts:
    - name: logs
      mountPath: /var/log
  - name: busybox
    image: busybox
    args: [/bin/sh, -c, 'tail -n+1 /var/log/big-corp-app.log']
    volumeMounts:
    - name: logs
      mountPath: /var/log
  volumes:
  - name: logs
    emptyDir: {}
# 删除旧的，创建新的sidecar
~ kubectl delete -f sidecar.yaml
~ kubectl apply -f sidecar.yaml
```

## 16.Top

```shell
# Task
From the pod label name=cpu-loader, find pods running high CPU wordloads and write the name of the pod consuming most CPU to the file /opt/KUTR00401/KUTR00401.txt(which already exists)
```

```shell
******* Prepare *******
~ kubectl config use-context k8s
******* Answer *******
~ kubectl top pod -A -l name=cpu-loader
# 将查到的占用CPU最大的pod的名字写到文件中，假设coredns-54d67798b7-hl8xc这个pod占用最高
~ echo "coredns-54d67798b7-hl8xc" >> /opt/KUTR00401/KUTR00401.txt
```

## 17.Node NotReady Check

```shell
# Task
A Kubernetes work node named wk8s-node-0 is in state NotReady. Investigate why this is the case, and perform any appropriate step to bring the node to a Ready state., ensuring that any changes are made permanent.

You can ssh to the failed node using: ssh wk8s-node-0
You can assume elevated privileges on the node with the following command: sudo -i
```

```shell
******* Prepare *******
~ kubectl config use-context wk8s
******* Answer *******
~ ssh wk8s-node-0
~ sudo -i
~ systemctl status kubelet
~ systemctl start kubelet
~ systemctl enable kubelet
```

## 1. RuntimeClass

[RuntimeClass解题参考](https://kubernetes.io/zh/docs/concepts/containers/runtime-class/#2-%E5%88%9B%E5%BB%BA%E7%9B%B8%E5%BA%94%E7%9A%84-runtimeclass-%E8%B5%84%E6%BA%90)

```bash
# context
This cluster uses containerd as CRI runtime. Containerd default runtime handler is runc.
Containerd has been prepared to support an additional runtime handler runsc(gVisor).
# Task
Create a RuntimeClass named untrusted using the prepared runtime handler named runsc.
Update all Pods in the namespace client to run on gvisor.
You can find a skeleton manifest file at /home/candidate/KSMV00301/runtime-class.yaml

# Prepare
$ kubectl config use-context KSMV00301
# Answer
$ vim runtimeclass.yaml 
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: untrusted
handler: runsc
$ kubectl apply -f runtimeclass.yaml
$ kubectl get pod -n client
# 假设查到的pod名为nginx
$ kubectl edit pod nginx -n client
...
spec:
  # spec下面加入下面一行
  runtimeClassName: untrusted
  ...

# 如果pod是deployment生成的，需要对应deployment中的spec:template:spec添加pod模板资源
# 例如查到的pod名为nginx-57cdfc6777-z2jkp, 对应的deployment为nginx
$ kubectl edit deploy nginx -n client
apiVersion: apps/v1
kind: Deployment
metadata:
...
spec:
...
  template:
  ...
    spec:
      # spec下面加入下面一行
      runtimeClassName: untrusted
      ...
```

## 2. ServiceAccount

[ServiceAccount解题参考](https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-service-account/#%E4%BD%BF%E7%94%A8%E9%BB%98%E8%AE%A4%E7%9A%84%E6%9C%8D%E5%8A%A1%E8%B4%A6%E6%88%B7%E8%AE%BF%E9%97%AE-api-%E6%9C%8D%E5%8A%A1%E5%99%A8)

```bash
# context
A Pod fails to run because of an incorrectly specified ServiceAcccount.
The name of ServiceAccount must end as "-sa".
The ServiceAccount must not have access to any API credentials.

# Task
1. Create a new ServiceAccount named backend-sa in the existing namespace prod.
2. You can create pod by using manifest file /home/candidate/KSCH00301/pod-manifest.yaml.
3. Finally, clean-up all unused ServiceAccount in namespace prod.

# Prepare
$ kubectl config use-context KSCH00301
# Answer
$ kubectl create sa backend-sa -n prod
$ kubectl edit sa -n prod backend-sa
apiVersion: v1
# 第二行（与apiVersion同级配置即可）添加下面禁止自动挂载API凭据配置
automountServiceAccountToken: false
...
$ vim /home/candidate/KSCH00301/pod-manifest.yaml
...
spec:
  # spec下加入一行
  serviceAccountName: backend-sa
  containers:
...
$ kubectl apply -f /home/candidate/KSCH00301/pod-manifest.yaml
$ kubectl get sa -n prod
# 查看哪些sa被pod使用了，删除没有被使用的sa
$ kubectl get pod -n prod -oyaml | grep -i serviceaccount
$ kubectl delete sa xxx -n prod
```

## 3. CIS Benchmark

```bash
# context
CIS Benchmark tool was run against the kubeadm-created cluster and found multiple issues that must be addressed immediately.
# Task
Fix all issues via configuration and restart theaffected components to ensure the new settings take effect.
Fix all of the following violations that were found against the API server:
Ensure that the 1.2.7 --authorization-mode FAIL argument is not set to AlwaysAllow
Ensure that the 1.2.8 --authorization-mode FAIL argument includes Node
Ensure that the 1.2.9 --authorization-mode FAIL argument includes RBAC

Fix all of the following violations that were found against the kubelet:
Ensure that the 4.2.1 anonymous-auth FAIL argument is set to false
Ensure that the 4.2.2 --authorization-mode FAIL argument is not set to AlwaysAllow

Use webhook authn/authz where possible.
Fix all of the following violations that were found against etcd:
Ensure that the 4.2.1 --client-cert-auth FAIL argument is set to true

# Prepare
$ kubectl config use-context KSCS00201
# Answer
# ssh到集群master节点，修正好这三项配置后，退出，ssh到node节点查看kubelet是否需要修正
$ ssh kscs00201-master
$ vim /etc/kubernetes/manifests/kube-apiserver.yaml
# 修改下面配置为Node,RBAC
- --authorization-mode=Node,RBAC
$ vim /etc/kubernetes/manifests/etcd.yaml
# 修改下面配置为true
- --client-cert-auth=true
# 修改下面两处配置
$ vim /var/lib/kubelet/config.yaml
...
authentication:
  anonymous:
    # 1.此处改为false
    enabled: false
...
authorization:
  # 2.此处改为Webhook
  mode: Webhook
...
$ systemctl restart kubelet
$ exit

# 退出master后ssh到node节点
$ ssh kscs00201-worker1
# 如果需要，则修改下面两处配置
$ vim /var/lib/kubelet/config.yaml
...
authentication:
  anonymous:
    # 1.此处改为false
    enabled: false
...
authorization:
  # 2.此处改为Webhook
  mode: Webhook
...
$ systemctl restart kubelet
$ exit
```

## 4. NetworkPolicy

[NetworkPolicy解题参考](https://kubernetes.io/zh/docs/concepts/services-networking/network-policies/#default-policies)

```bash
# context
A default-deny NetworkPolicy avoids to accidentally expose a Pod in a namespace that does not have any other NetworkPolicy defined.
# Task
Create a new default-deny NetworkPolicy named defaultdeny in the namespace production for all traffic of type Ingress.
The new NetworkPolicy must deny all lngress traffic in the namespace production.
Apply the newly created default-deny NetworkPolicy to all Pods running in namespace production.
You can find a skeleton manifest file at /home/candidate/KSCS00101/network-policy.yaml

# Prepare
$ kubectl config use-context KSCS00101
# Answer
$ vim defaultdeny.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: defaultdeny
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
$ kubectl apply -f defaultdeny.yaml
```

## 5. PodSecurityPolicy

[PodSecurityPolicy解题参考](https://kubernetes.io/zh/docs/concepts/policy/pod-security-policy/#set-up)

```bash
# context
A PodSecurityPolicy shall prevent the create on of privileged Pods in a specific namespace.
# Task
Create a new PodSecurityPolicy named deny-policy, which prevents the creation of privileged Pods.
Create a new ClusterRole named deny-access-role, which uses the newly created PodSecurityPolicy deny-policy.
Create a new ServiceAccount named psp-denial-sa in the existing namespace development.
Finally, create a new ClusterRoleBinding named deny-access-bind, which binds the newly created ClusterRole deny-access-role to the newly created ServiceAccount psp-denial-sa.

# Prepare
$ kubectl config use-context KSMV00102
# Answer
# 开启PSP功能
$ ssh ksmv00102-master
$ vim /etc/kubernetes/manifests/kube-apiserver.yaml
# 准入控制器加入PodSecurityPolicy
 - --enable-admission-plugins=NodeRestriction,PodSecurityPolicy
# 重启kubelet服务
$ systemctl restart kubelet
$ exit
$ vim psp.yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: deny-policy
spec:
  privileged: false  # Don't allow privileged pods!
  # The rest fills in some required fields.
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  runAsUser:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  volumes:
  - '*'
$ kubectl apply -f psp.yaml
$ kubectl create clusterrole deny-access-role --verb=use --resource=psp --resource-name=deny-policy
$ kubectl create sa psp-denial-sa -n development
$ kubectl create clusterrolebinding deny-access-bind --clusterrole=deny-access-role --serviceaccount=development:psp-denial-sa

# 如果创建的是rolebinding，需要加上namespace
$ kubectl create rolebinding deny-access-bind --clusterrole=deny-access-role --serviceaccount=development:psp-denial-sa -n development
```

## 6. RBAC

[RBAC解题参考](https://kubernetes.io/zh/docs/reference/access-authn-authz/rbac/#kubectl-create-role)

```bash
# context
A Role bound to a Pod serviceAccount grants overly permissive permissions.
Complete the following tasks to reduce the set of permissions.
# Task
Given an existing Pod named dev-pod running in the namespace monitoring.
Edit the existing Role bound to the Pod.
ServiceAccount service-account-web to only allow performing get operations, only on resources of type Pods.
Create a new Role named role-2 in the namespace monitoring, which only allows performing update operations, only on resources of type statefulsets.
Create a new RoleBinding named role-2-binding binding the newly created Role to the Pod ServiceAccount.
Don not delete the existing RoleBinding.

# Prepare
$ kubectl config use-context KSTR00103
# Answer
$ kubectl get role -n monitoring 
NAME       CREATED AT
web-role   2021-12-17T09:30:05Z
$ kubectl edit role -n monitoring web-role 
...
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
$ kubectl create role role-2 -n monitoring --verb=update --resource=statefulsets
$ kubectl create rolebinding role-2-binding -n monitoring --role=role-2 --serviceaccount=monitoring:service-account-web
```

## 7. 审计日志

[审计日志解题参考](https://kubernetes.io/zh/docs/tasks/debug-application-cluster/audit/#audit-policy)

```bash
# Task
Enable audit logs in the cluster.To do so, enable the log backend, and ensurethat:
 1. logs are stored at /var/log/kubernetes/audit-logs.txt
 2. log files are retained for 30 days
 3. at maximum, a number of 10 auditlog files are retained
A basic policy is provided at /etc/kubernetes/logpolicy/sample-policy.yaml. it only specifies what not to log.
The base policy is located on the cluster master node.
Edit and extend the basic policy to log:
 1. Cronjobs changes at RequestResponse level
 2. the request body of persistentvolumes changes in the namespace front-apps
 3. ConfigMap and Secret changes in all namespaces at the Metadata level
Also, add a catch-all ruie to log all otherrequests at the Metadata level.
Don not forget to apply the modifiedpolicy.

# Prepare
$ kubectl config use-context KSRS00602
# Answer
$ ssh ksrs00602-master
$ vim /etc/kubernetes/logpolicy/sample-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
# Don't generate audit events for all requests in RequestReceived stage.
omitStages:
  - "RequestReceived"
rules:
  # Log pod changes at RequestResponse level
  - level: RequestResponse
    resources:
    - group: ""
      resources: ["cronjobs"]
  # Log the request body of configmap changes in kube-system.
  - level: Request
    resources:
    - group: ""
      resources: ["persistentvolumes"]
    namespaces: ["front-apps"]
  # Log configmap and secret changes in all other namespaces at the Metadata level.
  - level: Metadata
    resources:
    - group: ""
      resources: ["secrets", "configmaps"]
  # A catch-all rule to log all other requests at the Metadata level.
  - level: Metadata
    omitStages:
      - "RequestReceived"
$ vim /etc/kubernetes/manifests/kube-apiserver.yaml
...
spec:
  containers:
  - command:
    - kube-apiserver
    # 添加下面四行
    - --audit-policy-file=/etc/kubernetes/logpolicy/sample-policy.yaml
    - --audit-log-path=/var/log/kubernetes/audit-logs.txt
    - --audit-log-maxage=30
    - --audit-log-maxbackup=10
    ...
# 重启kubelet
$ systemctl restart kubelet
# 可以查看一下日志输出，如果报错查看一下数据卷是否挂载，/etc/kubernetes/manifests/kube-apiserver.yaml文件最后
# 没挂载的话用参考链接中后端日志挂载数据卷并配置hostPath
$ tail -1 /var/log/kubernetes/audit-logs.txt
$ exit
```

## 8. Secret

[Secret解题参考](https://kubernetes.io/zh/docs/concepts/configuration/secret/#using-secrets-as-files-from-a-pod)
[Kubectl创建secret参考](https://kubernetes.io/zh/docs/tasks/configmap-secret/managing-secret-using-kubectl/)

```bash
# Task
Retrieve the content of the existing secret named db1-test in the namespace monitoring.
Store the username field in a file named /home/candidate/user.txt , and the password field in a file named /home/candidate/old-password.txt
You must create both files; they do not exist yet.
Do not use/modify the created files in the following steps, create new temporaryfiles if needed.
Create a new secret named dev-mark in the namespace monitoring, with the following content:
  username : production-instance
  password : aVJdk7NSjk
Finally, create a new Pod that has access to the secret dev-mark via a volume:
名称            内容
pod name        secret-pod
namespace       monitoring
container name  test-secret-container
image           redis
volume name     secret-volume
mount path      /etc/test-secret

# Prepare
$ kubectl config use-context KSMV00201
# Answer
$ kubectl get secrets -n monitoring db1-test -oyaml
apiVersion: v1
data:
  password: cGFzcw==
  username: YWRtaW4=
...
# 解密后存放到指定文件
$ echo "cGFzcw==" | base64 -d > /home/candidate/old-password.txt
$ echo "YWRtaW4=" | base64 -d > /home/candidate/user.txt
$ kubectl create secret generic dev-mark -n monitoring --from-literal=username=production-instance --from-literal=password=aVJdk7NSjk
$ vim secret-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
  namespace: monitoring
spec:
  containers:
  - name: test-secret-container
    image: redis
    volumeMounts:
    - name: secret-volume
      mountPath: "/etc/test-secret"
  volumes:
  - name: secret-volume
    secret:
      secretName: dev-mark
$ kubectl apply -f secret-pod.yaml
```

## 9. Dockerfile

```bash
# Task
Analyze and edit the given Dockerfile (based on the ubuntu:16.04 image) /home/candidate/KSSC00301/Dockerfile fixing two instructions present in the file being prominent security/best-practice issues.
Analyze and edit the given manifest file /home/candidate/KSSC00301/deployment.yaml fixing two fields present in the file being prominent security/best-practice issues.
You can use normal user(nobody),ID is 65535. 

# Prepare
$ kubectl config use-context KSSC00301
# Answer
$ vim /home/candidate/KSSC00301/Dockerfile
# root改为nobody
USER nobody
USER nobody
$ vim /home/candidate/KSSC00301/deployment.yaml
# 对应内容True|False修改为下面两处
'privileged': False,'readOnlyRootFilesystem': True,
```

## 10. Stateless && Immutable(无状态和不可变)

```bash
# context
lt is best-practice to design containers to best teless and immutable.
# Task
lnspect Pods running in namespace development and delete any Pod that is either not stateless or not immutable.
Using the following strict interpretation of stateless and immutable:
Pods being able to store data inside containers must be treated as not stateless.
You do not have to worry whether data is actually stored inside containers or not already.
Pods being configured to be privileged in any way must be treated as potentially not stateless and not immutable.

# Prepare
$ kubectl config use-context KSMV00201
# Answer
$ kubectl get po -n development -oyaml | grep "privileged"
# privileged: true
$ kubectl delete po -n develoment xxx --force
# pod不多的话可以edit看一下是否有特权配置和挂载nfs,hostPath等
```

## 11. NetworkPolicy

[NetworkPolicy解题参考](https://kubernetes.io/zh/docs/concepts/services-networking/network-policies/#networkpolicy-resource)

```bash
# Task
Create a NetworkPolicy named pod-restriction torestrict access to Pod products-service running in namespace development.
Only allow the following Pods to connect to Pod products-service :
Pods in the namespace qa
Pods with label environment: staging, in any namespace
Make sure to apply the NetworkPolicy.
You can find a skelet on manifest file at /home/candidate/KSSH00301/network-policy.yaml

# Prepare
$ kubectl config use-context KSSH00301
# Answer
# 查看namespace qa的标签
root@k8s-master:~/cks# kubectl get ns qa --show-labels
NAME   STATUS   AGE   LABELS
qa     Active   41s   kubernetes.io/metadata.name=qa
# 查看Pod products-service的标签
root@k8s-master:~/cks# kubectl get po products-service -n development --show-labels
NAME               READY   STATUS    RESTARTS   AGE   LABELS
products-service   1/1     Running   0          73s   run=products-service
# 编写yaml文件
root@k8s-master:~/cks# vim network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: pod-restriction
  namespace: development
spec:
  podSelector:
    matchLabels:
      run: products-service
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: qa
    - podSelector:
        matchLabels:
          environment: testing
root@k8s-master:~/cks# kubectl apply -f network-policy.yaml
```

## 12. ImagePolicyWebhook

```bash
# context
A container image scanner is set up on the cluster, but it is not yet fully integrated into the cluster configuration. When complete, the container image scanner shall scan for and reject the use of vulnerable images.
# Task
You have to complete the entire task on the cluster master node, where all services and files have been prepared and placed.
Given an incomplete configuration in directory /etc/kubernetes/epconfig and a functional container image scanner with HTTPS endpoint http://wakanda.local:8082/image_policy:
1. Enable the necessary plugins to create an image policy
2. Validate the control configuration and change it to an implicit deny
3. Edit the configuration to point t the provided HTTPS endpoint correctly.
Finally , test if the configuration is working by trying to deploy the vulnerable resource /root/KSSC00202/configuration-test.yaml
You can find the container image scanner log file at /var/log/imagepolicy/acme.log

# Prepare
$ kubectl config use-context KSSC00202
# Answer
$ ssh kssc00202-master
$ vim /etc/kubernetes/epconfig/admission_configuration.json
'defaultAllow': false # 改成 false
$ vim /etc/kubernetes/epconfig/kubeconfig.yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
 certificate-authority: /etc/kubernetes/pki/server.crt
 # 添加server配置
 server: https://wakanda.local:8082/image_policy
...
# 配置挂载，考试时一般会提前配置好
  - mountPath: /etc/kubernetes/epconfig
    name: epconfig
hostNetwork: true
...
volumes:
- name: epconfig
  hostPath:
    path: /etc/kubernetes/epconfig
# 添加ImagePolicyWebhook准入控制插件以及配置文件
$ vim /etc/kubernetes/manifests/kube-apiserver.yaml
- --enable-admission-plugins=NodeRestriction,ImagePolicyWebhook
- --admission-control-configfile=/etc/kubernetes/epconfig/admission_configuration.json
$ systemctl restart kubelet
$ exit
```

## 13. Trivy

```bash
# Task
Use the Trivy open-source container scanner to detect images with severe vulnerabilities used by Pods in the namespace kamino.
Look for images with High or Critical severity vulnerabilities,and delete the Pods that use those images.
Trivy is pre-installed on the cluster master node only; it is not available on the base system or the worker nodes. You will have to connect to the cluster master node to use Trivy.

# Prepare
$ kubectl config use-context KSSC00401
# Answer
$ ssh kssc00401-master
$ kubectl get po -n kamino
$ kubectl get po -n kamino -oyaml | grep image:
# 检测镜像，以nginx为例
$ trivy nginx | grep -E "HIGH|CRITICAL"
$ kubectl delete pod -n kamino nginx
$ exit
```

## 14. Admission

```bash
# context
kubeadm was used to create the cluster used in this task.
# Task
Reconfigure and restart the cluster Kubernetes APl server to ensure that only authenticated and authorized REST requests are allowed.
authorization-mode=Node,RBAC admission-plugins=NodeRestriction
Delete ClusterRoleBinding of user system:anonymous

# Prepare
$ kubectl config use-context KSCH00101
# Answer
$ ssh ksch00101-master
$ vim /etc/kubernetes/manifests/kube-apiserver.yaml
- --authorization-mode=Node,RBAC
- --enable-admission-plugins=NodeRestriction
$ systemctl restart kubelet
$ kubectl delete clusterrolebinding system:anonymous
$ exit
```

## 15. AppArmor

```bash
# Context
AppArmor is enabled on the cluster worker node. An AppArmor profile is prepared, but not enforced yet.
You may use your browser to open one additional tab to access theAppArmor documentation.
# Task
On the cluster worker node, enforce the prepared AppArmor profile located at /etc/apparmor.d/nginx_apparmor
Edit the prepared manifest file located at /home/candidate/KSSH00401/nginx-deploy.yaml to apply the AppArmor profile.
Finally, apply the manifest file and create the pod specified in it.

# Prepare
$ kubectl config use-context KSSH00401
# Answer
$ ssh kssh00401-worker1
# 查看profile名称为nginx-profile-1
$ cat /etc/apparmor.d/nginx_apparmor
...
profile nginx-profile-1 flags=(attach_disconnected) {
...
# 加载文件
$ apparmor_parser /etc/apparmor.d/nginx_apparmor
# 确认加载成功
$ apparmor_status | grep nginx
   nginx-profile-1
$ exit
# 退出后添加/home/candidate/KSSH00401/nginx-deploy.yaml文件中的annotation元数据后创建Pod
$ vim /home/candidate/KSSH00401/nginx-deploy.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-deploy
  # 添加下面两行，注意修改kubernetes.io/后面的名称为spec中的容器名nginx-deploy，localhost/后面名称改为profile名称nginx-profile-1
  annotations:
    container.apparmor.security.beta.kubernetes.io/nginx-deploy: localhost/nginx-profile-1
spec:
 containers:
 - name: nginx-deploy
...
$ kubectl apply -f /home/candidate/KSSH00401/nginx-deploy.yaml
```
