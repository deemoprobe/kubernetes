# 解决External-IP为Pending的问题

说明: 由于使用kubeadm创建的Kubernetes集群是没有集成`LoadBalancer`组件(像 AWS/googlecloud/阿里云这些云服务是集成好的), 所以在创建Nginx-Ingress时使用LoadBalancer会发生`Nginx-Ingress Service External-IP为Pending的问题`

为了解决这种问题, 使集群暴露的外部IP(External-IP)能被外界访问,需要在集群中安装配置一下MetalLB.

> MetalLB 是一个负载均衡器,专门解决裸金属Kubernetes 集群(Bare Metal Kubernetes Clusters)中无法使用 LoadBalancer 类型服务的痛点。 MetalLB 使用标准化的路由协议,以便裸金属Kubernetes 集群上的外部服务也尽可能地工作,MetalLB 能够帮助你在裸金属 Kubernetes 集群中创建 LoadBalancer 类型的 Kubernetes 服务

所谓裸金属Kubernetes 集群(Bare Metal Kubernetes Clusters): 即高性能的Kubernetes集群. 可联想到裸金属服务器(Bare Metal Server, BMS)-兼具虚拟机弹性和物理机性能的计算类服务器, 有物理机的性能又兼具虚拟机的轻便.

```shell
# 1. 安装MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/metallb.yaml
# On first install only
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"

# 2. 配置IP范围的ConfigMap
[root@k8s-master manifests]# cat metallbcm.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 172.42.42.100-172.42.42.120 #Update this with your Nodes IP range 
[root@k8s-master manifests]# kubectl apply -f metallbcm.yaml 
configmap/config created

# 3. 创建服务以获得外部 IP, 这里我创建的是Nginx-Ingress服务
# 安装MetalLB前
[root@k8s-master manifests]# kubectl get svc
NAME                                                     TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
nginx-ingress-nginx-ingress-controller                   LoadBalancer   192.168.120.98   <pending>     80:31051/TCP,443:30469/TCP   28m
nginx-ingress-nginx-ingress-controller-default-backend   ClusterIP      192.168.240.37   <none>        80/TCP                       28m
# 安装MetalLB后, EXTERNAL-IP以根据配置的IP范围分配好
[root@k8s-master manifests]# kubectl get svc
NAME                                                     TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)                      AGE
nginx-ingress-nginx-ingress-controller                   LoadBalancer   192.168.120.98   172.42.42.101   80:31051/TCP,443:30469/TCP   29m
nginx-ingress-nginx-ingress-controller-default-backend   ClusterIP      192.168.240.37   <none>          80/TCP                       29m
```
