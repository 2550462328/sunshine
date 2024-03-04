
![img](http://pcc.huitogo.club/4e109e7b7463a340d70a2265b196d8fc)针对这个集群架构来聊一聊kafka？



#### **为什么同一个Topic可以分发在多个Broker？**

若干个 Broker 组成一个集群（Cluster），其中集群内某个 Broker 会成为集群控制器（Cluster Controller），它负责管理集群，包括分配分区到 Broker、监控 Broker 故障等。在集群内，一个分区由一个 Broker 负责，这个 Broker 也称为这个分区的 Leader；当然一个分区可以被复制到多个 Broker 上来实现冗余，这样当存在 Broker 故障时可以将其分区重新分配到其他 Broker 来负责。



#### **为什么一个Topic下这么多Partition（多分区）？**

提供比较好的并发能力（负载均衡）。

如果我们把所有同类的消息都塞入到一个“中心”队列中，势必缺少可伸缩性，无论是生产者/消费者数目的增加，还是消息数量的增加，都可能耗尽系统的性能或存储。



我们使用一个生活中的例子来说明：现在 A 城市生产的某商品需要运输到 B 城市，走的是公路，那么单通道的高速公路不论是在「A 城市商品增多」还是「现在 C 城市也要往 B 城市运输东西」这样的情况下都会出现「吞吐量不足」的问题。所以我们现在引入**分区（Partition）**的概念，类似“允许多修几条道”的方式对我们的主题完成了水平扩展。



#### **为什么一个Partition还有这么多的replica（多副本）？**

提高了消息存储的安全性, 提高了容灾能力



#### **在主节点故障后每个副本都可以竞选主节点吗？**

在 Kafka 中并不是所有的副本都能被拿来替代主副本，所以在 Kafka 的Leader 节点中维护着一个 ISR（In sync Replicas）集合，翻译过来也叫正在同步中集合，在这个集合中的需要满足两个条件:

- 1、节点必须和 Zookeeper 保持连接。
- 2、在同步的过程中这个副本不能落后主副本太多。

另外还有个 AR（Assigned Replicas）用来标识副本的全集，OSR 用来表示由于落后被剔除的副本集合，所以公式如下：

- ISR = Leader + 没有落后太多的副本。
- AR = OSR + ISR 。



#### **Zookeeper在kafka中起到了什么作用？**

我们先看一下在kafka启动的时候zookeeper会为我们保存什么信息？

![img](http://pcc.huitogo.club/0a7a1fb30c97bd685cc506cbb4736421)



可以看到它为我们保存了broker信息、broker里面的topic信息，topic下的partition信息



所以可以看出zookeeper做了这么几件事

1. Broker 在 ZooKeeper 中的注册。

2. Topic 在 ZooKeeper 中的注册。

3. Consumer 在 ZooKeeper 中的注册。

4. Producer 负载均衡。

   > 主要指的是，Producer 从 Zookeeper 拉取 Topic 元数据，从而能够将消息发送负载均衡到对应 Topic 的分区中。

5. Consumer 负载均衡。

6. 记录消费进度 Offset 。

   > Kafka 已推荐将 consumer 的 Offset 信息保存在 Kafka 内部的 Topic 中。

7. 记录 Partition 与 Consumer 的关系。



其实，总结起来，就是两类功能：

- Broker、Producer、Consumer 和 Zookeeper 的交互。

  > 对应 1、2、3、5 。

- 相应的状态存储到 Zookeeper 中。

  > 对应 4、6、7 。



#### **kafka多集群之间怎么通信？**

随着业务发展，我们往往需要多集群，通常处于下面几个原因：

- 基于数据的隔离；
- 基于安全的隔离；
- 多数据中心（容灾）



当构建多个数据中心时，往往需要实现消息互通。举个例子，假如用户修改了个人资料，那么后续的请求无论被哪个数据中心处理，这个更新需要反映出来。又或者，多个数据中心的数据需要汇总到一个总控中心来做数据分析。

上面说的分区复制冗余机制只适用于同一个 Kafka 集群内部，对于多个 Kafka 集群消息同步可以使用 Kafka 提供的 **MirrorMaker** 工具。本质上来说，MirrorMaker 只是一个 Kafka 消费者和生产者，并使用一个队列连接起来而已。它从一个集群中消费消息，然后往另一个集群生产消息。



#### **Partition 与消费模型**

Partition 中的消息可以被（不同的 Consumer Group）多次消费，那 Partition中被消费的消息是何时删除的？ Partition 又是如何知道一个 Consumer Group 当前消费的位置呢？



无论消息是否被消费，除非消息到期 Partition 从不删除消息。例如设置保留时间为 2 天，则消息发布 2 天内任何 Group 都可以消费，2 天后，消息自动被删除。 Partition 会为每个 Consumer Group 保存一个偏移量，记录 Group 消费到的位置。 如下图：

![img](http://pcc.huitogo.club/fb198b077d9fabc457a60a062fb7a5b0)