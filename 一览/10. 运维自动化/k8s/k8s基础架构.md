**基本架构**

![](https://pcc.huitogo.club/k8s9.png)

K8S 是属于主从设备模型（Master-Slave 架构），即有 Master 节点负责核心的调度、管理和运维，Slave 节点则在执行用户的程序。Master Node 和 Worker Node 组成了 K8S 集群，同一个集群可能存在多个 Master Node 和 Worker Node。



Master Node都有哪些组件：

1）**API Server**。K8S 的请求入口服务。

- 为k8s控制平面和数据存储提供API接口
- 集群控制入口，所有的客户端（如：kubectl）或其他应用集成都必须通过 kube-apiserver API 交互
- 认证和授权、请求校验、准入控制



2）**Scheduler**：K8S 所有 Worker Node 的调度器。

- 负责将 Pods 指派到合适的集群Node节点上，通过 watch 机制发现 Pod 和 Node
- 预选：为Pod选出能够承载当前Pod的节点。输入是所有节点，输出是满足预选条件的节点。
- 优选：选出最优节点。输入是预选阶段筛选出的节点，优选会根据优先策略为通过预选的Nodes进行打分排名，选择得分最高的Node。



3）**Controller Manager**。K8S 所有 Worker Node 的监控器。

- 核心组件控制循环的主守护进程，用于调节声明式资源状态。 通过 API Server watch集群的资源， 并尝试将资源当前状态更改为期望状态。
- 包含多个内置资源控制器：如 Node Controller、Namespace Controller、Deployment Controller等



4）**etcd**。K8S 的存储服务。

- k8s集群数据库
- 一个可靠的key-value分布式存储系统
- 存储资源对象
- 仅 API Server 才具备读写权限，其他组件必须通过 API Server 的接口才能读写数据。



Worker Node都有哪些组件：

1）**Kubelet**。Worker Node 的监视器，以及与 Master Node 的通讯器。Kubelet 是 Master Node 安插在 Worker Node 上的“眼线”，它会定期向 Worker Node 汇报自己 Node 上运行的服务的状态，并接受来自 Master Node 的指示采取调整措施。



2）**Kube-Proxy**。K8S 的网络代理。

- 为 Service 执行流量转发、为 Pod 实现负载均衡。
- 常用的网络代理模式：iptables 和 ipvs



3）**Container Runtime**。Worker Node 的运行环境。即安装了容器化所需的软件环境确保容器化程序能够跑起来，比如 Docker Engine。大白话就是帮忙装好了 Docker 运行环境。



4）**Logging Layer**。K8S 的监控状态收集器。私以为称呼为 Monitor 可能更合适？Logging Layer 负责采集 Node 上所有服务的 CPU、内存、磁盘、网络等监控项信息。



5）**Add-Ons**。K8S 管理运维 Worker Node 的插件组件。有些文章认为 Worker Node 只有三大组件，不包含 Add-On，但笔者认为 K8S 系统提供了 Add-On 机制，让用户可以扩展更多定制化功能，是很不错的亮点。



**总结来看，K8S 的 Master Node 具备：请求入口管理（API Server），Worker Node 调度（Scheduler），监控和自动调节（Controller Manager），以及存储功能（etcd）；而 K8S 的 Worker Node 具备：状态和监控收集（Kubelet），网络和负载均衡（Kube-Proxy）、保障容器化运行环境（Container Runtime）、以及定制化功能（Add-Ons）。**