# Kubernetes监控之Metrics-Server

> 使用Helm部署K8S资源指标获取工具: metrics-server

项目地址: <https://github.com/kubernetes-sigs/metrics-server>

```bash
# 默认情况下下面资源指标是无法获取的
[root@k8s-master ingress]# kubectl top node
error: Metrics API not available
[root@k8s-master ingress]# kubectl top pod
error: Metrics API not available
```

获取metrics-server-amd64镜像

```bash
# 在集群所有节点都需要执行
docker pull registry.cn-beijing.aliyuncs.com/google_registry/metrics-server-amd64:v0.3.6
docker tag  registry.cn-beijing.aliyuncs.com/google_registry/metrics-server-amd64:v0.3.6 k8s.gcr.io/metrics-server-amd64:v0.3.6
docker rmi  registry.cn-beijing.aliyuncs.com/google_registry/metrics-server-amd64:v0.3.6
```

配置metrics-server.yaml文件

```bash
[root@k8s-master kubernetes]# mkdir monitor;cd monitor
[root@k8s-master monitor]# vi metrics-server.yaml 
args:
- --logtostderr
- --kubelet-insecure-tls
- --kubelet-preferred-address-types=InternalIP
```

通过helm部署metrics-server

```bash
# 添加官方最新的Helm仓库
[root@k8s-master monitor]# helm repo add stable https://charts.helm.sh/stable
"stable" has been added to your repositories
[root@k8s-master monitor]# helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "apphub" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈Happy Helming!⎈
[root@k8s-master monitor]# helm repo list
NAME    URL                          
apphub  https://apphub.aliyuncs.com  
stable  https://charts.helm.sh/stable
# 先搜一波
[root@k8s-master monitor]# helm search repo stable | grep metrics-server
stable/metrics-server                   2.11.4          0.3.6                   DEPRECATED - Metrics Server is a cluster-wide a...
# 安装
[root@k8s-master monitor]# helm install metrics-server stable/metrics-server --namespace kube-system -f metrics-server.yaml
NAME: metrics-server
LAST DEPLOYED: Tue Jan 12 15:14:57 2021
NAMESPACE: kube-system
STATUS: deployed
REVISION: 1
NOTES:
The metric server has been deployed. 

In a few minutes you should be able to list metrics using the following
command:

  kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes"
# 查看Metrics-Server相关资源
[root@k8s-master monitor]# helm list --namespace=kube-system
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
metrics-server  kube-system     1               2021-01-12 15:14:57.092114352 +0800 CST deployed        metrics-server-2.11.4   0.3.6      
[root@k8s-master monitor]# kubectl get deploy -A | grep 'metrics-server'
kube-system      metrics-server                                           1/1     1            1           99s
[root@k8s-master monitor]# kubectl get pod -A | grep 'metrics-server'
kube-system      metrics-server-68cc74d4d-7cj44                                    1/1     Running            0          77s
# 获取node节点资源占用情况
[root@k8s-master monitor]# kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes"
{"kind":"NodeMetricsList","apiVersion":"metrics.k8s.io/v1beta1","metadata":{"selfLink":"/apis/metrics.k8s.io/v1beta1/nodes"},"items":[{"metadata":{"name":"k8s-master","selfLink":"/apis/metrics.k8s.io/v1beta1/nodes/k8s-master","creationTimestamp":"2021-01-12T07:18:02Z"},"timestamp":"2021-01-12T07:17:40Z","window":"30s","usage":{"cpu":"1395283488n","memory":"1246356Ki"}},{"metadata":{"name":"k8s-node1","selfLink":"/apis/metrics.k8s.io/v1beta1/nodes/k8s-node1","creationTimestamp":"2021-01-12T07:18:02Z"},"timestamp":"2021-01-12T07:17:37Z","window":"30s","usage":{"cpu":"224689267n","memory":"997016Ki"}}]}
# 查看node节点资源使用情况
[root@k8s-master monitor]# kubectl top node
NAME         CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
k8s-master   1396m        69%    1217Mi          70%       
k8s-node1    225m         11%    973Mi           36%  
# 查看所有Pod的资源使用情况
[root@k8s-master monitor]# kubectl top po -A
NAMESPACE        NAME                                                              CPU(cores)   MEMORY(bytes)   
default          busybox-k8s-master                                                0m           0Mi             
default          high-priority                                                     1m           1Mi             
default          look-svc-env-k8s-master                                           0m           0Mi             
default          myapp-deploy1-6c468d6b6c-nr28d                                    0m           1Mi             
default          myapp-deploy1-6c468d6b6c-w69sz                                    0m           1Mi             
default          nginx-k8s-master                                                  0m           1Mi             
default          pod-dns-k8s-master                                                0m           1Mi             
dev              my-nginx-7c4ff94949-9c9m2                                         0m           1Mi             
dev              myapp-deploy1-6c468d6b6c-cq44z                                    0m           1Mi             
dev              myapp-deploy1-6c468d6b6c-mzqwd                                    0m           1Mi             
dev              myapp-deploy2-5fffdcccd5-25qz4                                    0m           1Mi             
dev              myapp-deploy2-5fffdcccd5-vpcj9                                    0m           1Mi             
dev              nginx-ingress-nginx-ingress-controller-667cb64f9c-zqg2q           4m           100Mi           
dev              nginx-ingress-nginx-ingress-controller-default-backend-7ccrfv9m   1m           2Mi             
kube-system      calico-kube-controllers-5c6f6b67db-4gwqx                          11m          18Mi            
kube-system      calico-node-g5r6h                                                 65m          96Mi            
kube-system      calico-node-tw24f                                                 76m          61Mi            
kube-system      coredns-6d56c8448f-m92h8                                          14m          20Mi            
kube-system      coredns-6d56c8448f-wh66t                                          6m           11Mi            
kube-system      etcd-k8s-master                                                   49m          60Mi            
kube-system      kube-apiserver-k8s-master                                         161m         472Mi           
kube-system      kube-controller-manager-k8s-master                                54m          70Mi            
kube-system      kube-proxy-2ld6t                                                  3m           14Mi            
kube-system      kube-proxy-bx9jg                                                  1m           18Mi            
kube-system      kube-scheduler-k8s-master                                         13m          25Mi            
kube-system      metrics-server-68cc74d4d-7cj44                                    4m           11Mi            
metallb-system   controller-fb659dc8-c9r7p                                         3m           7Mi             
metallb-system   speaker-dwl4w                                                     8m           13Mi            
metallb-system   speaker-ptc97                                                     10m          13Mi            
test             my-nginx-7c4ff94949-n5jh8                                         0m           3Mi             
test             nginxingress-nginx-ingress-controller-cddb9cd67-nddz9             4m           95Mi            
test             nginxingress-nginx-ingress-default-backend-6d7d9db4d7-dwl7n       1m           1Mi 
```
