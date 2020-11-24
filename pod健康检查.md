# Kubernetes pod健康检查

## LivenessProbe探针

Liveness 探测让用户可以自定义判断容器是否健康的条件。如果探测失败，Kubernetes 就会重启容器。

```shell
# 创建liveness.yaml
[root@k8s-master manifests]# vi liveness.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness
spec:
  restartPolicy: OnFailure
  containers:
  - name: liveness
    image: busybox
    args:
    - /bin/sh
    - -c
    - echo ok > /tmp/healthy; sleep 10; rm -rf /tmp/healthy; sleep 60
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 15
      timeoutSecond: 1

# 查看检查失败的日志以及后续操作
[root@k8s-master manifests]# kubectl describe po liveness
Events:
  Type     Reason     Age                 From               Message
  ----     ------     ----                ----               -------
  Normal   Scheduled  114s                default-scheduler  Successfully assigned default/liveness to k8s-node1
  Normal   Pulled     97s                 kubelet            Successfully pulled image "busybox" in 15.98593525s
  Warning  Unhealthy  55s (x3 over 75s)   kubelet            Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
  Normal   Killing    55s                 kubelet            Container liveness failed liveness probe, will be restarted
  Normal   Pulling    26s (x2 over 113s)  kubelet            Pulling image "busybox"
  Normal   Created    0s (x2 over 97s)    kubelet            Created container liveness
  Normal   Started    0s (x2 over 96s)    kubelet            Started container liveness
  Normal   Pulled     0s                  kubelet            Successfully pulled image "busybox" in 26.131183161s
```

说明：在pod运行后，将创建的/tmp/health文件10s后删除，LivenessProbe探针健康检查探测时间是15s，检查结果`Container liveness failed liveness probe`，然后容器会重启。
