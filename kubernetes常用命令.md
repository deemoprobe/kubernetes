# kubernetes常用命令总结

## 0. 常用

```shell
# 开放端口,将nginx服务的8080端口映射到本地30080端口
kubectl port-forward --address 0.0.0.0 pod/nginx-6799fc88d8-xj4c4 30080:8080
# pod容器内和本地传输文件
kubectl cp nginx:db2379282hbbdsv:/etc/fstab /tmp
```

## 1. kubectl apply

- 以文件或标准输入为准应用或更新资源

```shell
# 使用 example-service.yaml 中的定义创建服务。
kubectl apply -f example-service.yaml

# 使用 example-controller.yaml 中的定义创建 replication controller。
kubectl apply -f example-controller.yaml

# 使用 <directory> 路径下的任意 .yaml, .yml, 或 .json 文件 创建对象。
kubectl apply -f <directory>
```

## 2. kubectl get

- 列出一个或多个资源

```shell
# 以纯文本输出格式列出所有 pod。
kubectl get pods

# 以纯文本输出格式列出所有 pod，并包含附加信息(如节点名)。
kubectl get pods -o wide

# 以纯文本输出格式列出具有指定名称的副本控制器。提示：您可以使用别名 'rc' 缩短和替换 'replicationcontroller' 资源类型。
kubectl get replicationcontroller <rc-name>

# 以纯文本输出格式列出所有副本控制器和服务。
kubectl get rc,services

# 以纯文本输出格式列出所有守护程序集，包括未初始化的守护程序集。
kubectl get ds --include-uninitialized

# 列出在节点 server01 上运行的所有 pod
kubectl get pods --field-selector=spec.nodeName=server01

# 将单个 pod 的详细信息输出为 YAML 格式的对象：

kubectl get pod web-pod-13je7 -o yaml

```

## 3. kubectl describe

- 显示一个或多个资源的详细状态，默认情况下包括未初始化的资源

说明：  
kubectl get 命令通常用于检索同一资源类型的一个或多个资源。 它具有丰富的参数，允许您使用 -o 或 --output 参数自定义输出格式。您可以指定 -w 或 --watch 参数以开始观察特定对象的更新。 kubectl describe 命令更侧重于描述指定资源的许多相关方面。它可以调用对 API 服务器 的多个 API 调用来为用户构建视图。 例如，该 kubectl describe node 命令不仅检索有关节点的信息，还检索在其上运行的 pod 的摘要，为节点生成的事件等。

```shell
# 显示名称为 <node-name> 的节点的详细信息。
kubectl describe nodes <node-name>

# 显示名为 <pod-name> 的 pod 的详细信息。
kubectl describe pods/<pod-name>

# 显示由名为 <rc-name> 的副本控制器管理的所有 pod 的详细信息。
# 记住：副本控制器创建的任何 pod 都以复制控制器的名称为前缀。
kubectl describe pods <rc-name>

# 描述所有的 pod，不包括未初始化的 pod
kubectl describe pods --include-uninitialized=false

```

## 4. kubectl delete

- 从文件、stdin 或指定标签选择器、名称、资源选择器或资源中删除资源

```shell
# 使用 pod.yaml 文件中指定的类型和名称删除 pod。
kubectl delete -f pod.yaml

# 删除标签名= <label-name> 的所有 pod 和服务。
kubectl delete pods,services -l name=<label-name>

# 删除所有具有标签名称= <label-name> 的 pod 和服务，包括未初始化的那些。
kubectl delete pods,services -l name=<label-name> --include-uninitialized

# 删除所有 pod，包括未初始化的 pod。
kubectl delete pods --all

```

## 5. kubectl exec

- 对 pod 中的容器执行命令

```shell
# 从 pod <pod-name> 中获取运行 'date' 的输出。默认情况下，输出来自第一个容器。
kubectl exec <pod-name> date
  
# 运行输出 'date' 获取在容器的 <container-name> 中 pod <pod-name> 的输出。
kubectl exec <pod-name> -c <container-name> date

# 获取一个交互 TTY 并运行 /bin/bash <pod-name >。默认情况下，输出来自第一个容器。
kubectl exec -ti <pod-name> /bin/bash

```

## 6. kubectl logs

- 打印 Pod 中容器的日志

```shell
# 从 pod 返回日志快照。
kubectl logs <pod-name>

# 从 pod <pod-name> 开始流式传输日志。这类似于 'tail -f' Linux 命令。
kubectl logs -f <pod-name>
```

## 7. kubectl plugin

- 插件

```shell
# 用任何语言创建一个简单的插件，并为生成的可执行文件命名
# 以前缀 "kubectl-" 开始
cat ./kubectl-hello
#!/bin/bash

# 这个插件打印单词 "hello world"
echo "hello world"

# 我们的插件写好了，让我们把它变成可执行的
sudo chmod +x ./kubectl-hello

# 并将其移动到路径中的某个位置
sudo mv ./kubectl-hello /usr/local/bin

# 我们现在已经创建并"安装"了一个 kubectl 插件。
# 我们可以开始使用我们的插件，从 kubectl 调用它，就像它是一个常规命令一样
kubectl hello
hello world
# 我们可以"卸载"一个插件，只需从我们的路径中删除它
sudo rm /usr/local/bin/kubectl-hello

```

为了查看可用的所有 kubectl 插件，我们可以使用 kubectl plugin list 子命令：

```shell
kubectl plugin list
```

以下 kubectl-适配 的插件是可用的：

```shell
/usr/local/bin/kubectl-hello
/usr/local/bin/kubectl-foo
/usr/local/bin/kubectl-bar

# 这个指令也可以警告我们哪些插件
# 被运行，或是被其它插件覆盖了
# 例如
sudo chmod -x /usr/local/bin/kubectl-foo
kubectl plugin list
以下 kubectl-适配 的插件是可用的：

/usr/local/bin/kubectl-hello
/usr/local/bin/kubectl-foo
# 警告: /usr/local/bin/kubectl-foo 被识别为一个插件，但是它并不可以执行
/usr/local/bin/kubectl-bar

```

错误: 发现了一个插件警告
我们可以将插件视为在现有 kubectl 命令之上构建更复杂功能的一种方法：

```shell
cat ./kubectl-whoami
#!/bin/bash

# 这个插件借用 `kubectl config` 指令来输出
# 当前用户的信息，基于当前指定的 context
kubectl config view --template='{{ range .contexts }}{{ if eq .name "'$(kubectl config current-context)'" }}Current user: {{ .context.user }}{{ end }}{{ end }}'

```

运行上面的插件为我们提供了一个输出，其中包含我们 KUBECONFIG 文件中当前所选定上下文对应的用户：

```shell
# 使文件成为可执行的
sudo chmod +x ./kubectl-whoami

# 然后移动到我们的路径中
sudo mv ./kubectl-whoami /usr/local/bin

kubectl whoami
Current user: plugins-user
```
