![Eureka 集群](https://pcc.huitogo.club/z0/25e9704082444add1192bc69c87198e9)

- 作用：实现服务治理（服务注册与发现）
- 简介：Spring Cloud Eureka是Spring Cloud Netflix项目下的服务治理模块。
- 由两个组件组成：Eureka 服务端和 Eureka 客户端。
  - Eureka 服务端，用作服务注册中心，支持集群部署。各个服务启动后，会在Eureka Server中进行注册，这样Eureka Server的服务注册表中将会存储所有可用服务节点的信息，服务节点的信息可以在界面中直观的看到。
  - Eureka 客户端，是一个 Java 客户端，用来处理服务注册与发现。作为一个Java客户端，用于简化与Eureka Server的交互。Eureka Client内置一个 使用轮询负载算法的负载均衡器。服务启动后，Eureka Client将会向Eureka Server发送心跳更新服务，如果Eureka Server在多个心跳周期内没有接收到某个服务的心跳，Eureka Server将会从服务注册表中把这个服务节点移除(默认90秒)。



在应用启动时，Eureka 客户端向服务端注册自己的服务信息，同时将服务端的服务信息缓存到本地。客户端会和服务端周期性的进行心跳交互，以更新服务租约和服务信息。



Eureka 原理，整体如下图：

![Eureka 原理](https://pcc.huitogo.club/z0/80c74f1d7cb9fc2a416e7b61a055d778)



