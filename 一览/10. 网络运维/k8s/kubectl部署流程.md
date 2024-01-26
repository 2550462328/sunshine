#### 1. 部署流程

例如新部署nginx服务

首先要了解，一旦kubernetes环境启动之后，master和node都会将自身的信息存储到etcd数据库中。

1. 一个nginx服务的安装请求会首先被发送到master节点的apiServer组件。
2. apiServer组件会调用scheduler组件来决定到底应该把这个服务安装到哪个node节点上。在此时，它会从etcd中读取各个node节点的信息，然后按照一定的算法进行选择，并将结果告知apiServer。
3. apiServer调用controller-manager去调度Node节点安装nginx服务。
4. kubelet接收到指令后，会通知docker。
5. 然后由docker来启动一个nginx的pod。



pod是kubernetes的最小操作单元，容器必须跑在pod中。至此，一个nginx服务就运行了，如果需要访问nginx，就需要通过kube-proxy来对pod产生访问的代理，这样，外界用户就可以访问集群中的nginx服务了。

![](https://pcc.huitogo.club/k8s10.png)



#### 2. 创建pod流程

1. 客户端提交创建请求，可以通过API Server的Restful API，也可以使用kubectl命令行工具。支持的数据类型包括JSON和YAML。
2. API Server处理用户请求，存储Pod数据到etcd。
3. 调度器通过API Server查看未绑定的Pod。尝试为Pod分配主机。
4. 过滤主机 (调度预选)：调度器用一组规则过滤掉不符合要求的主机。比如Pod指定了所需要的资源量，那么可用资源比Pod需要的资源量少的主机会被过滤掉。
5. 主机打分(调度优选)：对第一步筛选出的符合要求的主机进行打分，在主机打分阶段，调度器会考虑一些整体优化策略，比如把容一个Replication Controller的副本分布到不同的主机上，使用最低负载的主机等。
6. 选择主机：选择打分最高的主机，进行binding操作，结果存储到etcd中。
7. kubelet根据调度结果执行Pod创建操作： 绑定成功后，scheduler会调用APIServer的API在etcd中创建一个boundpod对象，描述在一个工作节点上绑定运行的所有pod信息。运行在每个工作节点上的kubelet也会定期与etcd同步boundpod信息，一旦发现应该在该工作节点上运行的boundpod对象没有更新，则调用Docker API创建并启动pod内的容器。

![](https://pcc.huitogo.club/k8s11.png)



在整个生命周期中，Pod会出现5种状态（相位），分别如下：

- **挂起(Pending)**：apiserver已经创建了pod资源对象，但它尚未被调度完成或者仍处于下载镜像的过程中
- **运行中(Running)**：pod已经被调度至某节点，并且所有容器都已经被kubelet创建完成
- **成功(Succeeded)**：pod中的所有容器都已经成功终止并且不会被重启
- **失败(Failed)**：所有容器都已经终止，但至少有一个容器终止失败，即容器返回了非0值的退出状态
- **未知(Unknown)**：apiserver无法正常获取到pod对象的状态信息，通常由网络通信失败所导致