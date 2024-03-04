**云原生架构**

![截图.png](https://pcc.huitogo.club/k8s1.png)

 

**云原生分层**

1）容器(docker) 是云原生的核心基础组件，在云原生平台上，一切软件都被容器化。

2）k8s 提供了对容器的分布式编排和调度，从而完成了对一切被容器化的软件的分布式编排和调度。

3）除了 k8s 原始的 kubectl 命令，helm 提供了更便利的 k8s 集群的包管理。

例如，使用 helm 可以在 k8s 集群上安装 mysql, 安装python 服务程序。

当然，使用 helm 也可以在 k8s 集群上安装服务网格(ServiceMesh)的事实标准istio套装。

4）istio 提供了k8s之上的服务网格全功能代理，将分布式系统构架里的流量管理、安全控制、可观察性下层到云原生基础设施。

5）继续，使用 helm 可以在k8s内直接部署CI/CD软件，例如Jenkins，直接提供了高可用的CI/CD，完成应用程序的自动拉取，测试、打包和镜像构建上传，以及k8s集群里的滚动更新。

6）最后，跨云上的k8s基础设施配置，可以通过 Terraform 完成标准化、可移植的管理。



**k8s基础概念**

![截图.png](https://pcc.huitogo.club/k8s2.png)

- 节点（Node）：一个节点是一个运行 Kubernetes 中的主机。

- 容器组（Pod）：一个 Pod 对应于由若干容器组成的一个容器组，同个组内的容器共享一个存储卷(volume)。

- 容器组生命周期（pos-states）：包含所有容器状态集合，包括容器组状态类型，容器组生命周期，事件，重启策略，以及 replication controllers。

- Replication Controllers：主要负责指定数量的 pod 在同一时间一起运行。

- 服务（services）：一个 Kubernetes 服务是容器组逻辑的高级抽象，同时也对外提供访问容器组的策略。

- 卷（volumes）：一个卷就是一个目录，容器对其有访问权限。

- 标签（labels）：标签是用来连接一组对象的，比如容器组。标签可以被用来组织和选择子对象。

- 接口权限（accessing_the_api）：端口，IP 地址和代理的防火墙规则。

- web 界面（ux）：用户可以通过 web 界面操作 Kubernetes。

- 命令行操作（cli）：kubectl命令。


 

**什么是K8S？**

K8S 是负责自动化运维管理多个 Docker 程序的集群。自动完成服务的部署、更新、卸载和扩容、缩容！

 

**什么是 Pod ？**

Pod是可以在 Kubernetes 中创建和管理的、最小的可部署的计算单元。

所谓最小 xx 单位”要么就是事物的衡量标准单位，要么就是资源的闭包、集合。前者比如长度米、时间秒；后者比如一个”进程“是存储和计算的闭包，一个”线程“是 CPU 资源（包括寄存器、ALU 等）的闭包。

Pod 就是 K8S 中一个服务的闭包。Pod 可以被理解成一群可以共享网络、存储和计算资源的容器化服务的集合。

同一个 Pod 之间的 Container 可以通过 localhost 互相访问，并且可以挂载 Pod 内所有的数据卷；但是不同的 Pod 之间的 Container 不能用 localhost 访问，也不能挂载其他 Pod 的数据卷。

![截图.png](https://pcc.huitogo.club/k8s3.png)

 

K8S 中所有的对象都通过 yaml 来表示。



一个最简单的Pod的yaml：

```
apiVersion: v1

kind: Pod

metadata:

 name: memory-demo

 namespace: mem-example

spec:

 containers:

 - name: memory-demo-ctr

  image: polinux/stress

  resources:

   limits:

     memory: "200Mi"

   requests:

     memory: "100Mi"

  command: ["stress"]

  args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]

  volumeMounts:

  - name: redis-storage

   mountPath: /data/redis

 volumes:

 - name: redis-storage

  emptyDir: {}
```

- apiVersion记录 K8S 的 API Server 版本，现在看到的都是v1，用户不用管。

- kind记录该 yaml 的对象，比如这是一份 Pod 的 yaml 配置文件，那么值内容就是Pod。

- metadata记录了 Pod 自身的元数据，比如这个 Pod 的名字、这个 Pod 属于哪个 namespace（命名空间的概念，后文会详述，暂时理解为“同一个命名空间内的对象互相可见”）。

- spec记录了 Pod 内部所有的资源的详细信息，看懂这个很重要：

- containers记录了 Pod 内的容器信息，containers包括了：name容器名，image容器的镜像地址，resources容器需要的 CPU、内存、GPU 等资源，command容器的入口命令，args容器的入口参数，volumeMounts容器要挂载的 Pod 数据卷等。可以看到，上述这些信息都是启动容器的必要和必需的信息。

- volumes记录了 Pod 内的数据卷信息


 

**什么是Volume 数据卷？**

**Volume 数据卷是需要手动 mount 的磁盘，是 Pod 内部的磁盘资源。**

volume 是 K8S 的对象，对应一个实体的数据卷；而 volumeMounts 只是 container 的挂载点，对应 container 的其中一个参数。但是，volumeMounts 依赖于 volume，只有当 Pod 内有 volume 资源的时候，该 Pod 内部的 container 才可能有 volumeMounts。

 

**什么是Deployment？**

一个 Deployment 控制器为 Pods 和 ReplicaSets 提供声明式的更新能力。

你负责描述 Deployment 中的 目标状态，而 Deployment 控制器以受控速率更改实际状态， 使其变为期望状态。你可以定义 Deployment 以创建新的 ReplicaSet，或删除现有 Deployment，并通过新的 Deployment 收养其资源。



翻译一下：**Deployment 的作用是管理和控制 Pod 和 ReplicaSet**，**管控它们运行在用户期望的状态中**。哎，打个形象的比喻，**Deployment 就是包工头**，主要负责监督底下的工人 Pod 干活，确保每时每刻有用户要求数量的 Pod 在工作。如果一旦发现某个工人 Pod 不行了，就赶紧新拉一个 Pod 过来替换它。

 

除了Deployment管理pods外，还有

- StatefulSet：通常适用于有状态服务，让你能够运行一个或者多个以某种方式跟踪应用状态的 Pods。

- DaemonSet：定义提供节点本地支撑设施的 Pods。这些 Pods 可能对于你的集群的运维是 非常重要的，例如作为网络链接的辅助工具或者作为网络 插件 的一部分等等。

- Job 和 CronJob：定义一些一直运行到结束并停止的任务。


 

**那什么是ReplicaSets？**

ReplicaSet 的目的是维护一组在任何时候都处于运行状态的 Pod 副本的稳定集合。 因此，它通常用来保证给定数量的、完全相同的 Pod 的可用性。



再来翻译下：ReplicaSet 的作用就是管理和控制 Pod，管控他们好好干活。但是，ReplicaSet 受控于 Deployment。形象来说，**ReplicaSet 就是总包工头手下的小包工头**。

![截图.png](https://pcc.huitogo.club/k8s4.png)



注意：从 K8S 使用者角度来看，用户会直接操作 Deployment 部署服务，而当 Deployment 被部署的时候，K8S 会自动生成要求的 ReplicaSet 和 Pod。在K8S 官方文档中也指出**用户只需要关心 Deployment 而不操心 ReplicaSet**。

 

**什么是Service?**

将运行在一组 Pods 上的应用程序公开为网络服务的抽象方法。

使用 Kubernetes，您无需修改应用程序即可使用不熟悉的服务发现机制。 Kubernetes 为 Pods 提供自己的 IP 地址，并为一组 Pod 提供相同的 DNS 名， 并且可以在它们之间进行负载均衡。

翻译下：K8S 中的服务（Service）并不是我们常说的“服务”的含义，而更像是网关层，是若干个 Pod 的流量入口、流量均衡器。

 

**为什么要 Service 呢?**

Kubernetes Pod 是有生命周期的。 它们可以被创建，而且销毁之后不会再启动。 如果您使用 Deployment 来运行您的应用程序，则它可以动态创建和销毁 Pod。

每个 Pod 都有自己的 IP 地址，但是在 Deployment 中，在同一时刻运行的 Pod 集合可能与稍后运行该应用程序的 Pod 集合不同。

这导致了一个问题： 如果一组 Pod（称为“后端”）为群集内的其他 Pod（称为“前端”）提供功能， 那么前端如何找出并跟踪要连接的 IP 地址，以便前端可以使用工作量的后端部分？

K8S 集群内的每一个 Pod 都有自己的 IP（是不是很类似一个 Pod 就是一台服务器，然而事实上是多个 Pod 存在于一台服务器上，只不过是 K8S 做了网络隔离），在 K8S 集群内部还有 DNS 等网络服务（一个 K8S 集群就如同管理了多区域的服务器，可以做复杂的网络拓扑）。

 

总结：**Service 是 K8S 服务的核心，屏蔽了服务细节，统一对外暴露服务接口，真正做到了“微服务”。**举个例子，我们的一个服务 A，部署了 3 个备份，也就是 3 个 Pod；对于用户来说，只需要关注一个 Service 的入口就可以，而不需要操心究竟应该请求哪一个 Pod。优势非常明显：一方面外部用户不需要感知因为 Pod 上服务的意外崩溃、K8S 重新拉起 Pod 而造成的 IP 变更，外部用户也不需要感知因升级、变更服务带来的 Pod 替换而造成的 IP 变化，另一方面，Service 还可以做流量负载均衡。

 

**网络通信**

1）同节点pod间通信

![截图.png](https://pcc.huitogo.club/k8s5.png)

每个pod都有自己的network namespace，通过一个 veth pair 将其连接到root namespace。 veth pair 是一对管道，一端在根网络中，另一端在Pod network namespace中。

 

2）跨节点pod通信

![截图.png](https://pcc.huitogo.club/k8s6.png)

**什么是Ingress ？**

上文我们了解到Service 主要负责 K8S 集群内部的网络拓扑。那么集群外部怎么访问集群内部呢？这个时候就需要 Ingress 了，官方文档中的解释是：

> Ingress 是对集群中服务的外部访问进行管理的 API 对象，典型的访问方式是 HTTP。
>
> Ingress 可以提供负载均衡、SSL 终结和基于名称的虚拟托管。
>



翻译一下：Ingress 是整个 K8S 集群的接入层，复杂集群内外通讯。

![截图.png](https://pcc.huitogo.club/k8s7.png)

 

**什么是namespace?**

Kubernetes 支持多个虚拟集群，它们底层依赖于同一个物理集群。 这些虚拟集群被称为名字空间。

翻译一下：namespace 是为了把一个 K8S 集群划分为若干个资源不可共享的虚拟集群而诞生的。也就是说，**可以通过在 K8S 集群内创建 namespace 来分隔资源和对象**。

![截图.png](https://pcc.huitogo.club/k8s8.png)