**什么是kubectl？**

Kubectl 是一个命令行接口，用于对 Kubernetes 集群运行命令。Kubectl 的配置文件在$HOME/.kube 目录。我们可以通过设置 KUBECONFIG 环境变量或设置命令参数--kubeconfig 来指定其他位置的 kubeconfig 文件。

也就是说，可以通过 kubectl 来操作 K8S 集群



基本语法

kubectl [command] [TYPE] [NAME] [flags]

- command：定要对一个或多个资源执行的操作，例如 create、get、describe、delete。
- TYPE：指定资源类型。资源类型不区分大小写，可以指定单数、复数或缩写形式。
- NAME：指定资源的名称。名称区分大小写。
- flags：指定可选的参数。例如，可以使用 `-s` 或 `-server` 参数指定 Kubernetes API 服务器的地址和端口。



**配置kubectl?**

第一步，必须准备好要连接/使用的 K8S 的配置文件，例如：

```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: thisisfakecertifcateauthoritydata00000000000
    server: https://1.2.3.4:1234
  name: cls-dev
contexts:
- context:
    cluster: cls-dev
    user: kubernetes-admin
  name: kubernetes-admin@test
current-context: kubernetes-admin@test
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    token: thisisfaketoken00000
```



解读如下：

1）clusters记录了 clusters（一个或多个 K8S 集群）信息：

- name是这个 cluster（K8S 集群）的名称代号
- server是这个 cluster（K8S 集群）的访问方式，一般为 IP+PORT
- certificate-authority-data是证书数据，只有当 cluster（K8S 集群）的连接方式是 https 时，为了安全起见需要证书数据

2）users记录了访问 cluster（K8S 集群）的账号信息：

- name是用户账号的名称代号
- user/token是用户的 token 认证方式，token 不是用户认证的唯一方式，其他还有账号+密码等。

3）contexts是上下文信息，包括了 cluster（K8S 集群）和访问 cluster（K8S 集群）的用户账号等信息：

- name是这个上下文的名称代号
- cluster是 cluster（K8S 集群）的名称代号
- user是访问 cluster（K8S 集群）的用户账号代号

4）current-context记录当前 kubectl 默认使用的上下文信息

5）kind和apiVersion都是固定值，用户不需要关心

6）preferences则是配置文件的其他设置信息，笔者没有使用过，暂时不提。



第二步，给 kubectl 配置上配置文件

有如下3种方式关联kubectl到配置文件

- --kubeconfig参数：每次执行 kubectl 的时候，都带上--kubeconfig=${CONFIG_PATH}。

- KUBECONFIG环境变量：使用环境变量KUBECONFIG把所有配置文件都记录下来

  ```
  export KUBECONFIG=$KUBECONFIG:${CONFIG_PATH}
  ```

- $HOME/.kube/config 配置文件：把配置文件的内容放到$HOME/.kube/config 内。



第三步：配置正确的上下文

如果配置文件只有一个 cluster 是没有任何问题的，但是对于有多个 cluster 怎么办呢？

```
# 列出所有上下文信息
kubectl config get-contexts
# 查看当前的上下文信息
kubectl config current-context
# 更改上下文信息
kubectl config use-context ${CONTEXT_NAME}
# 修改上下文的元素。比如可以修改用户账号、集群信息、连接到 K8S 后所在的 namespace
kubectl config set-context ${CONTEXT_NAME}|--current --${KEY}=${VALUE}
```

