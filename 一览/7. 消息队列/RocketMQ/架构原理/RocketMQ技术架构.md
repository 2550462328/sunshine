我们先看下RocketMQ的工作流程：

![整体流程](https://pcc.huitogo.club/z0/r412351231.png)

1. 启动 **Namesrv**，Namesrv起 来后监听端口，等待 Broker、Producer、Consumer 连上来，相当于一个路由控制中心。

2. **Broker** 启动，跟**所有的** Namesrv 保持长连接，定时发送心跳包。

   > 心跳包中，包含当前 Broker 信息(IP+端口等)以及存储所有 Topic 信息。
   > 注册成功后，Namesrv 集群中就有 Topic 跟 Broker 的映射关系。

3. 收发消息前，先创建 Topic 。创建 Topic 时，需要指定该 Topic 要存储在 哪些 Broker上。也可以在发送消息时自动创建Topic。

4. **Producer** 发送消息。

   > 启动时，先跟 Namesrv 集群中的其中一台建立长连接，并从Namesrv 中获取当前发送的 Topic 存在哪些 Broker 上，然后跟对应的 Broker 建立长连接，直接向 Broker 发消息。

5. **Consumer** 消费消息。

   > Consumer 跟 Producer 类似。跟其中一台 Namesrv 建立长连接，获取当前订阅 Topic 存在哪些 Broker 上，然后直接跟 Broker 建立连接通道，开始消费消息。



再附上一张架构图：

![img](http://pcc.huitogo.club/53baeb70d388de042f7347d137b9d35e)



RocketMQ架构上主要分为四部分，如上图所示：

#### 1. Producer

- **获得 Topic-Broker 的映射关系**。

  - Producer 启动时，也需要指定 Namesrv 的地址，从 Namesrv 集群中选一台建立长连接。如果该 Namesrv 宕机，会自动连其他 Namesrv ，直到有可用的 Namesrv 为止。
  - 生产者每 30 秒从 Namesrv 获取 Topic 跟 Broker 的映射关系，更新到本地内存中。然后再跟 Topic 涉及的所有 Broker 建立长连接，每隔 30 秒发一次心跳。
  - 在 Broker 端也会每 10 秒扫描一次当前注册的 Producer ，如果发现某个 Producer 超过 2 分钟都没有发心跳，则断开连接。

- **生产者端的负载均衡**。

  - 生产者发送时，会自动轮询当前所有可发送的broker，一条消息发送成功，下次换另外一个broker发送，以达到消息平均落到所有的broker上。

    > 这里需要注意一点：假如某个 Broker 宕机，意味生产者最长需要 30 秒才能感知到。在这期间会向宕机的 Broker 发送消息。当一条消息发送到某个 Broker 失败后，会自动再重发 2 次，假如还是发送失败，则抛出发送失败异常。
    >
    > 客户端里会自动轮询另外一个 Broker 重新发送，这个对于用户是透明的。



#### 2. Consumer

- 获得 Topic-Broker 的映射关系。

  - Consumer 启动时需要指定 Namesrv 地址，与其中一个 Namesrv 建立长连接。消费者每隔 30 秒从 Namesrv 获取所有Topic 的最新队列情况，这意味着某个 Broker 如果宕机，客户端最多要 30 秒才能感知。连接建立后，从 Namesrv 中获取当前消费 Topic 所涉及的 Broker，直连 Broker 。
  - Consumer 跟 Broker 是长连接，会每隔 30 秒发心跳信息到Broker 。Broker 端每 10 秒检查一次当前存活的 Consumer ，若发现某个 Consumer 2 分钟内没有心跳，就断开与该 Consumer 的连接，并且向该消费组的其他实例发送通知，触发该消费者集群的负载均衡。

- **消费者端的负载均衡**。根据消费者的消费模式不同，负载均衡方式也不同。

  > 消费者有两种消费模式：集群消费和广播消费。
  >
  > - 集群消费：一个 Topic 可以由同一个消费这分组( Consumer Group )下所有消费者分担消费。
  >   具体例子：假如 TopicA 有 6 个队列，某个消费者分组起了 2 个消费者实例，那么每个消费者负责消费 3 个队列。如果再增加一个消费者分组相同消费者实例，即当前共有 3 个消费者同时消费 6 个队列，那每个消费者负责 2 个队列的消费。
  > - 广播消费：每个消费者消费 Topic 下的所有队列。

  

***Q1：消费者获取消息有哪些模式？***

实际上**获取实际消息内容都是采用pull的模式**，push也只是pull的一种变种，它只是提醒消费者有新的消息需要拉取。

1. **PushConsumer**

   推送模式（虽然 RocketMQ 使用的是长轮询）的消费者。消息的能及时被消费。使用非常简单，内部已处理如线程池消费、流控、负载均衡、异常处理等等的各种场景。

2. **PullConsumer**

   拉取模式的消费者。应用主动控制拉取的时机，怎么拉取，怎么消费等。主动权更高。但要自己处理各种场景。



#### 3. NameServer

NameServer是一个非常简单的Topic路由注册中心，其角色类似Dubbo中的zookeeper，支持Broker的动态注册与发现。主要包括两个功能：

- Broker管理，NameServer接受Broker集群的注册信息并且保存下来作为路由信息的基本数据。然后提供心跳检测机制，检查Broker是否还存活；
- 路由信息管理，每个NameServer将保存关于Broker集群的整个路由信息和用于客户端查询的队列信息。然后Producer和Conumser通过NameServer就可以知道整个Broker集群的路由信息，从而进行消息的投递和消费。NameServer通常也是集群的方式部署，各实例间相互不进行信息通讯。



Broker是向每一台NameServer注册自己的路由信息，所以每一个NameServer实例上面都保存一份完整的路由信息。

当某个NameServer因某种原因下线了，Broker仍然可以向其它NameServer同步其路由信息，Producer,Consumer仍然可以动态感知Broker的路由的信息。

即使整个 Namesrv 集群宕机，已经正常工作的 Producer、Consumer、Broker 仍然能正常工作，但新起的 Producer、Consumer、Broker 就无法工作。



#### 4. BrokerServer

BrokerServer的特性有：

- **高并发读写服务**。Broker的高并发读写主要是依靠以下两点：

  - 消息顺序写，所有 Topic 数据同时只会写一个文件，一个文件满1G ，再写新文件，真正的顺序写盘，使得发消息 TPS 大幅提高。
  - 消息随机读，RocketMQ 尽可能让读命中系统 Pagecache ，因为操作系统访问 Pagecache 时，即使只访问 1K 的消息，系统也会提前预读出更多的数据，在下次读时就可能命中 Pagecache ，减少 IO 操作。

- **负载均衡与动态伸缩**

  - 负载均衡：Broker 上存 Topic 信息，Topic 由多个队列组成，队列会平均分散在多个 Broker 上，而 Producer 的发送机制保证消息尽量平均分布到所有队列中，最终效果就是所有消息都平均落在每个 Broker 上。

  - 动态伸缩能力（非顺序消息）：Broker 的伸缩性体现在两个维度：Topic、Broker。

    - Topic 维度：假如一个 Topic 的消息量特别大，但集群水位压力还是很低，就可以扩大该 Topic 的队列数， Topic 的队列数跟发送、消费速度成正比。

      > Topic 的队列数一旦扩大，就无法很方便的缩小。因为，生产者和消费者都是基于相同的队列数来处理。
      > 如果真的想要缩小，只能新建一个 Topic ，然后使用它。
      > 不过，Topic 的队列数，也不存在什么影响的，淡定。

    - Broker 维度：如果集群水位很高了，需要扩容，直接加机器部署 Broker 就可以。Broker 启动后向 Namesrv 注册，Producer、Consumer 通过 Namesrv 发现新Broker，立即跟该 Broker 直连，收发消息。

      > 新增的 Broker 想要下线，想要下线也比较麻烦，暂时没特别好的方案。大体的前提是，消费者消费完该 Broker 的消息，生产者不往这个 Broker 发送消息。

-  **高可用 & 高可靠**

  - 高可用：集群部署时一般都为主备，备机实时从主机同步消息，如果其中一个主机宕机，备机提供消费服务，但不提供写服务。

  - 高可靠：所有发往 Broker 的消息，有同步刷盘和异步刷盘机制。

    - 同步刷盘时，消息写入物理文件才会返回成功。

    - 异步刷盘时，只有机器宕机，才会产生消息丢失，Broker 挂掉可能会发生，但是机器宕机崩溃是很少发生的，除非突然断电。

      > 如果 Broker 挂掉，未同步到硬盘的消息，还在 Pagecache 中呆着。

-  **Broker 与 Namesrv 的心跳机制**

  - 单个 Broker 跟所有 Namesrv 保持心跳请求，心跳间隔为30秒，心跳请求中包括当前 Broker 所有的 Topic 信息。
  - Namesrv 会反查 Broker 的心跳信息，如果某个 Broker 在 2 分钟之内都没有心跳，则认为该 Broker 下线，调整 Topic 跟 Broker 的对应关系。但此时 Namesrv 不会主动通知Producer、Consumer 有 Broker 宕机。也就说，只能等 Producer、Consumer 下次定时拉取 Topic 信息的时候，才会发现有 Broker 宕机。



Broker主要包含了以下几个重要子模块：

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



**5）RocketMq节点间通信使用的是Netty。**