# Kubernetes进阶之问题总结

- 问题1：K8S安装完成之后，某一个节点的Swap内存打开了，对与K8S系统的影响是啥？

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

- 问题2：如果K8S的某个node 出现了NotReady的状态,怎么排查

- 问题3：`kubectl get cs`查看组件为unhealthy状态

排查方案：

流程：
1.kubectl get pods -A查看所有pod是否正常
2.检查kube-scheduler和kube-controller-manager组件配置是否禁用了非安全端口
vim /etc/kubernetes/manifests/kube-scheduler.yaml
vim /etc/kubernetes/manifests/kube-controller-manager.yaml
删除`--port=0`这一行
3.重启kubelet
systemctl restart kubelet
