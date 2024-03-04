#### 1. 集群工作流程

![img](http://pcc.huitogo.club/5ed4beaca1ade8a61b10321bca1020d8)



RocketMQ 网络部署特点

1. NameServer是一个几乎无状态节点，可集群部署，节点之间无任何信息同步。
2. Broker部署相对复杂，Broker分为Master与Slave，一个Master可以对应多个Slave，但是一个Slave只能对应一个Master，Master与Slave 的对应关系通过指定相同的BrokerName，不同的BrokerId 来定义，BrokerId为0表示Master，非0表示Slave。Master也可以部署多个。每个Broker与NameServer集群中的所有节点建立长连接，定时注册Topic信息到所有NameServer。 注意：当前RocketMQ版本在部署架构上支持一Master多Slave，但**只有BrokerId=1的从服务器才会参与消息的读负载**。
3. Producer与NameServer集群中的其中一个节点（随机选择）建立长连接，定期从NameServer获取Topic路由信息，并**向提供Topic 服务的Master建立长连接**，且定时向Master发送心跳。Producer完全无状态，可集群部署。
4. Consumer与NameServer集群中的其中一个节点（随机选择）建立长连接，定期从NameServer获取Topic路由信息，并向提供Topic服务的Master、Slave建立长连接，且定时向Master、Slave发送心跳。Consumer既可以从Master订阅消息，也可以从Slave订阅消息，消费者在向Master拉取消息时，Master服务器会根据拉取偏移量与最大偏移量的距离（判断是否读老消息，产生读I/O），以及从服务器是否可读等因素建议下一次是从Master还是Slave拉取。



结合部署架构图，描述集群工作流程：

🦅 **Producer**

- 1、Producer 自身在应用中，所以无需考虑高可用。
- 2、Producer 配置多个 Namesrv 列表，从而保证 Producer 和 Namesrv 的连接高可用。并且，会从 Namesrv 定时拉取最新的 Topic 信息。
- 3、Producer 会和所有 Broker 直连，在发送消息时，会选择一个 Broker 进行发送。如果发送失败，则会使用另外一个 Broker 。
- 4、Producer 会定时向 Broker 心跳，证明其存活。而 Broker 会定时检测，判断是否有 Producer 异常下线。

🦅 **Consumer**

- 1、Consumer 需要部署多个节点，以保证 Consumer 自身的高可用。当相同消费者分组中有新的 Consumer 上线，或者老的 Consumer 下线，会重新分配 Topic 的 Queue 到目前消费分组的 Consumer 们。
- 2、Consumer 配置多个 Namesrv 列表，从而保证 Consumer 和 Namesrv 的连接高可用。并且，会从 Consumer 定时拉取最新的 Topic 信息。
- 3、Consumer 会和所有 Broker 直连，消费相应分配到的 Queue 的消息。如果消费失败，则会发回消息到 Broker 中。
- 4、Consumer 会定时向 Broker 心跳，证明其存活。而 Broker 会定时检测，判断是否有 Consumer 异常下线。

🦅 **Namesrv**

- 1、Namesrv 需要部署多个节点，以保证 Namesrv 的高可用。
- 2、Namesrv 本身是无状态，不产生数据的存储，是通过 Broker 心跳将 Topic 信息同步到 Namesrv 中。
- 3、多个 Namesrv 之间不会有数据的同步，是通过 Broker 向多个 Namesrv 多写。

🦅 **Broker**

- 1、多个 Broker 可以形成一个 Broker 分组。每个 Broker 分组存在一个 Master 和多个 Slave 节点。
  - Master 节点，可提供读和写功能。Slave 节点，可提供读功能。
  - Master 节点会不断发送新的 CommitLog 给 Slave节点。Slave 节点不断上报本地的 CommitLog 已经同步到的位置给 Master 节点。
  - Slave 节点会从 Master 节点拉取消费进度、Topic 配置等等。
- 2、多个 Broker 分组，形成 Broker 集群。
  - Broker 集群和集群之间，不存在通信与数据同步。
- 3、Broker 可以配置同步刷盘或异步刷盘，根据消息的持久化的可靠性来配置。



#### 2.Broker节点消息复制

在Borker 主从模式下消息复制有**同步复制**和**异步复制**两种。

- 同步复制： 也叫 “同步双写”，也就是说，只有消息同步双写到主从结点上时才返回写入成功 。
- 异步复制： 消息写入主节点之后就直接返回写入成功 。



异步复制会不会也像异步刷盘那样影响消息的可靠性呢？

答案是**不会**的，因为两者就是不同的概念，对于**消息可靠性是通过不同的刷盘策略保证的，而像异步同步复制策略仅仅是影响到了 可用性** 。为什么呢？因为主Broker宕机后，从Broker还是可以提供读消息服务的，因为是异步同步，所以会有一部分消息没有同步过来，会有**短暂的消息不一致情况**，但是在**主Broker重启后从节点那部分未来得及复制的消息还会继续复制**。



这个可用性问题可以**通过添加多个主从Broker解决**。

但是如果某个业务（比如订单消息）只存放在一个Broker主从中呢？



RocketMQ 中采用了 **Dledger** 解决这个问题。他要求在写入消息的时候，要求**至少消息复制到半数以上的节点之后**，才给客⼾端返回写⼊成功，并且它是⽀持通过选举来动态切换主节点的。

但是Dledger 也不是个完美的方案，至少在 Dledger **选举过程中是无法提供服务的**，而且他**必须要使用三个节点或以上**，如果多数节点同时挂掉他也是无法保证可用性的，而且要求消息复制半数以上节点的**效率**和直接异步复制还是有一定的差距的，毕竟它和同步复制也只差半数以下的节点。