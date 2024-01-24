#### 1. 集群工作流程

![img](http://pcc.huitogo.club/5ed4beaca1ade8a61b10321bca1020d8)



RocketMQ 网络部署特点

1. NameServer是一个几乎无状态节点，可集群部署，节点之间无任何信息同步。
2. Broker部署相对复杂，Broker分为Master与Slave，一个Master可以对应多个Slave，但是一个Slave只能对应一个Master，Master与Slave 的对应关系通过指定相同的BrokerName，不同的BrokerId 来定义，BrokerId为0表示Master，非0表示Slave。Master也可以部署多个。每个Broker与NameServer集群中的所有节点建立长连接，定时注册Topic信息到所有NameServer。 注意：当前RocketMQ版本在部署架构上支持一Master多Slave，但只有BrokerId=1的从服务器才会参与消息的读负载。
3. Producer与NameServer集群中的其中一个节点（随机选择）建立长连接，定期从NameServer获取Topic路由信息，并向提供Topic 服务的Master建立长连接，且定时向Master发送心跳。Producer完全无状态，可集群部署。
4. Consumer与NameServer集群中的其中一个节点（随机选择）建立长连接，定期从NameServer获取Topic路由信息，并向提供Topic服务的Master、Slave建立长连接，且定时向Master、Slave发送心跳。Consumer既可以从Master订阅消息，也可以从Slave订阅消息，消费者在向Master拉取消息时，Master服务器会根据拉取偏移量与最大偏移量的距离（判断是否读老消息，产生读I/O），以及从服务器是否可读等因素建议下一次是从Master还是Slave拉取。



结合部署架构图，描述集群工作流程：

1. 启动NameServer，NameServer起来后监听端口，等待Broker、Producer、Consumer连上来，相当于一个路由控制中心。
2. Broker启动，跟所有的NameServer保持长连接，定时发送心跳包。心跳包中包含当前Broker信息(IP+端口等)以及存储所有Topic信息。注册成功后，NameServer集群中就有Topic跟Broker的映射关系。
3. 收发消息前，先创建Topic，创建Topic时需要指定该Topic要存储在哪些Broker上，也可以在发送消息时自动创建Topic。
4. Producer发送消息，启动时先跟NameServer集群中的其中一台建立长连接，并从NameServer中获取当前发送的Topic存在哪些Broker上，轮询从队列列表中选择一个队列，然后与队列所在的Broker建立长连接从而向Broker发消息。
5. Consumer跟Producer类似，跟其中一台NameServer建立长连接，获取当前订阅Topic存在哪些Broker上，然后直接跟Broker建立连接通道，开始消费消息。



#### 2. RocketMq中Broker节点消息复制

在Borker 主从模式下消息复制有**同步复制**和**异步复制**两种。

- 同步复制： 也叫 “同步双写”，也就是说，只有消息同步双写到主从结点上时才返回写入成功 。
- 异步复制： 消息写入主节点之后就直接返回写入成功 。



异步复制会不会也像异步刷盘那样影响消息的可靠性呢？

答案是**不会**的，因为两者就是不同的概念，对于**消息可靠性是通过不同的刷盘策略保证的，\**而像\**异步同步复制策略仅仅是影响到了 可用性** 。为什么呢？因为主Broker宕机后，从Broker还是可以提供读消息服务的，因为是异步同步，所以会有一部分消息没有同步过来，会有**短暂的消息不一致情况**，但是在**主Broker重启后从节点那部分未来得及复制的消息还会继续复制**。



这个可用性问题可以**通过添加多个主从Broker解决**。

但是如果某个业务（比如订单消息）只存放在一个Broker主从中呢？



RocketMQ 中采用了 **Dledger** 解决这个问题。他要求在写入消息的时候，要求**至少消息复制到半数以上的节点之后**，才给客⼾端返回写⼊成功，并且它是⽀持通过选举来动态切换主节点的。

但是Dledger 也不是个完美的方案，至少在 Dledger **选举过程中是无法提供服务的**，而且他**必须要使用三个节点或以上**，如果多数节点同时挂掉他也是无法保证可用性的，而且要求消息复制半数以上节点的**效率**和直接异步复制还是有一定的差距的，毕竟它和同步复制也只差半数以下的节点。