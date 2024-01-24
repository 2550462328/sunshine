![img](http://pcc.huitogo.club/53baeb70d388de042f7347d137b9d35e)



RocketMQ架构上主要分为四部分，如上图所示：

#### 1. Producer

消息发布的角色，支持分布式集群方式部署。Producer通过MQ的负载均衡模块选择相应的Broker集群队列进行消息投递，投递的过程支持快速失败并且低延迟。



#### 2. Consumer

消息消费的角色，支持分布式集群方式部署。支持以push推，pull拉两种模式对消息进行消费。同时也支持集群方式和广播方式的消费，它提供实时消息订阅机制，可以满足大多数用户的需求。



#### 3. NameServer

NameServer是一个非常简单的Topic路由注册中心，其角色类似Dubbo中的zookeeper，支持Broker的动态注册与发现。主要包括两个功能：Broker管理，NameServer接受Broker集群的注册信息并且保存下来作为路由信息的基本数据。然后提供心跳检测机制，检查Broker是否还存活；路由信息管理，每个NameServer将保存关于Broker集群的整个路由信息和用于客户端查询的队列信息。然后Producer和Conumser通过NameServer就可以知道整个Broker集群的路由信息，从而进行消息的投递和消费。NameServer通常也是集群的方式部署，各实例间相互不进行信息通讯。Broker是向每一台NameServer注册自己的路由信息，所以每一个NameServer实例上面都保存一份完整的路由信息。当某个NameServer因某种原因下线了，Broker仍然可以向其它NameServer同步其路由信息，Producer,Consumer仍然可以动态感知Broker的路由的信息。



#### 4. BrokerServer

Broker主要负责消息的存储、投递和查询以及服务高可用保证，为了实现这些功能，Broker包含了以下几个重要子模块：

- Remoting Module：整个Broker的实体，负责处理来自clients端的请求。
- Client Manager：负责管理客户端(Producer/Consumer)和维护Consumer的Topic订阅信息
- Store Service：提供方便简单的API接口处理消息存储到物理硬盘和查询功能。
- HA Service：高可用服务，提供Master Broker 和 Slave Broker之间的数据同步功能。
- Index Service：根据特定的Message key对投递到Broker的消息进行索引服务，以提供消息的快速查询。

![img](http://pcc.huitogo.club/852c89dfc811fa743a7e3891012787d1)



#### 5. 总结

1）我们的 Broker 做了**集群并且还进行了主从部署** ，由于消息分布在各个 Broker 上，一旦某个 Broker 宕机，则该Broker 上的消息读写都会受到影响。所以 Rocketmq 提供了 master/slave 的结构， salve 定时从 master 同步数据(同步刷盘或者异步刷盘)，如果 master 宕机，则 **slave 提供消费服务，但是不能写入消息**。



2）为了保证 **HA** ，我们的 NameServer 也做了集群部署，但是请注意它是 **去中心化** 的。也就意味着它没有主节点，你可以很明显地看出 NameServer 的所有节点是没有进行 Info Replicate 的，在 RocketMQ 中是通过 单个Broker和所有NameServer保持长连接 ，并且在每隔30秒 Broker 会向所有 Nameserver 发送心跳，心跳包含了自身的 Topic 配置信息，这个步骤就对应这上面的 Routing Info 。



正是因为NameServer的这种简单处理，可能导致NameServer之间的**数据不一致**。

而且当有新的服务器加入时，NameServer 并不会立马通知到 Produer，而是由 Produer 定时去请求 NameServer 获取最新的 Broker/Consumer 信息。



3）在生产者需要向 Broker 发送消息的时候，需要先从 NameServer 获取关于 Broker 的路由信息，然后通过 **轮询** 的方法去向每个队列中生产数据以达到 **负载均衡** 的效果。



4）消费者通过 NameServer 获取所有 Broker 的路由信息后，向 Broker 发送 Pull 请求来获取消息数据。Consumer 可以以两种模式启动—— **广播（Broadcast）和集群（Cluster）广播模式下，一条消息会发送给 同一个消费组中的所有消费者 ，集群模式下消息只会发送给一个消费者。**在广播模式下消费位置是每个消费者自己维护的，而在集群模式下是由Broker统一管理的。



**5）rocketMq节点间通信使用的是Netty。**