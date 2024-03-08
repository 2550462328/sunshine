为了保证高可用，最好是以集群形态来部署 ZooKeeper，这样只要集群中大部分机器是可用的（能够容忍一定的机器故障），通常 3 台服务器就可以构成一个 ZooKeeper 集群了。



![img](http://pcc.huitogo.club/516e0353b13b9202bc8c61bc37897916)



上述server集群间通过 **ZAB 协议**（ZooKeeper Atomic Broadcast）来保持数据的一致性。



#### 1. 集群角色

在 ZooKeeper 中引入了 Leader、Follower 和 Observer 三种角色。

![img](http://pcc.huitogo.club/536fbeb7be97ef9a982e7b1440bad1e8)



其中

- **Leader**： 一个ZooKeeper集群同一时间只会有一个实际工作的Leader，它会发起并维护与各Follwer及Observer间的心跳。所有的写操作必须要通过Leader完成再由Leader将写操作广播给其它服务器。
- **Follower**： 一个ZooKeeper集群可能同时存在多个Follower，它会响应Leader的心跳。Follower可直接处理并返回客户端的读请求，同时会将写请求转发给Leader处理，并且负责在Leader处理写请求时对请求进行投票。
- **Observer**：角色与Follower类似，但是无投票权。



#### 2. 集群服务器状态

- LOOKING ：寻找 Leader。

- LEADING ：Leader 状态，对应的节点为 Leader。

- FOLLOWING ：Follower 状态，对应的节点为 Follower。

- OBSERVING ：Observer 状态，对应节点为 Observer，该节点不参与 Leader 选举。

  

#### 3. 选举流程

当 Leader 服务器出现网络中断、崩溃退出与重启等异常情况时，就会进入 Leader 选举过程，这个过程会选举产生新的 Leader 服务器。



这个过程大致是这样的：

1. Leader election（选举阶段）：节点在一开始都处于选举阶段，只要有一个节点得到超半数节点的票数，它就可以当选准 leader。
2. Discovery（发现阶段）：在这个阶段，followers 跟准 leader 进行通信，同步 followers 最近接收的事务提议。
3. Synchronization（同步阶段）：同步阶段主要是利用 leader 前一阶段获得的最新提议历史，同步集群中所有的副本。同步完成之后 准 leader 才会成为真正的 leader。
4. Broadcast（广播阶段）：到了这个阶段，ZooKeeper 集群才能正式对外提供事务服务，并且 leader 可以进行消息广播。同时如果有新的节点加入，还需要对新节点进行同步。



##### 3.1 集群启动选举过程

**1）初始投票给自己**

集群刚启动时，所有服务器的logicClock都为1，zxid都为0。各服务器初始化后，都投票给自己，并将自己的一票存入自己的票箱，如下图所示。

![img](http://pcc.huitogo.club/3418f0d1621cbb2e3119c6f620d42e61)



在上图中，(1, 1, 0)第一位数代表投出该选票的服务器的logicClock，第二位数代表被推荐的服务器的myid，第三位代表被推荐的服务器的最大的zxid。由于该步骤中所有选票都投给自己，所以第二位的myid即是自己的myid，第三位的zxid即是自己的zxid。

此时各自的票箱中只有自己投给自己的一票。



**2）更新选票**

服务器收到外部投票后，进行选票PK，相应更新自己的选票并广播出去，并将合适的选票存入自己的票箱，如下图所示。

![img](http://pcc.huitogo.club/6921c8c00c29ec8d90cd0c2262e5be96)



服务器1收到服务器2的选票（1, 2, 0）和服务器3的选票（1, 3, 0）后，由于所有的logicClock都相等，所有的zxid都相等，因此根据myid判断应该将自己的选票按照服务器3的选票更新为（1, 3, 0），并将自己的票箱全部清空，再将服务器3的选票与自己的选票存入自己的票箱，接着将自己更新后的选票广播出去。此时服务器1票箱内的选票为(1, 3)，(3, 3)。

同理，服务器2收到服务器3的选票后也将自己的选票更新为（1, 3, 0）并存入票箱然后广播。此时服务器2票箱内的选票为(2, 3)，(3, ,3)。

服务器3根据上述规则，无须更新选票，自身的票箱内选票仍为（3, 3）。

服务器1与服务器2更新后的选票广播出去后，由于三个服务器最新选票都相同，最后三者的票箱内都包含三张投给服务器3的选票。



**3）根据选票确定角色**

根据上述选票，三个服务器一致认为此时服务器3应该是Leader。因此服务器1和2都进入FOLLOWING状态，而服务器3进入LEADING状态。之后Leader发起并维护与Follower间的心跳。

![img](http://pcc.huitogo.club/e367b236eca5576a699fb8b9a2d6a996)



##### 3.2 Follow宕机选举过程

**1）Follower重启投票给自己**

Follower重启，或者发生网络分区后找不到Leader，会进入LOOKING状态并发起新的一轮投票。

![img](http://pcc.huitogo.club/19a5d13bf13912908f4a025ab38af84b)



**2）发现已有Leader后成为Follower**

服务器3收到服务器1的投票后，将自己的状态LEADING以及选票返回给服务器1。服务器2收到服务器1的投票后，将自己的状态FOLLOWING及选票返回给服务器1。此时服务器1知道服务器3是Leader，并且通过服务器2与服务器3的选票可以确定服务器3确实得到了超过半数的选票。因此服务器1进入FOLLOWING状态。

![img](http://pcc.huitogo.club/9f94b44fb48b23d7a54605b99ed4ad13)



##### 3.3 Leader宕机选举

**1）Follower发起新投票**

Leader（服务器3）宕机后，Follower（服务器1和2）发现Leader不工作了，因此进入LOOKING状态并发起新的一轮投票，并且都将票投给自己。

![img](http://pcc.huitogo.club/58b3a4ab2e734df14a85955f5fad2370)



**2）广播更新选票**

服务器1和2根据外部投票确定是否要更新自身的选票。这里有两种情况：

1. 服务器1和2的zxid相同。例如在服务器3宕机前服务器1与2完全与之同步。此时选票的更新主要取决于myid的大小
2. 服务器1和2的zxid不同。在旧Leader宕机之前，其所主导的写操作，只需过半服务器确认即可，而不需所有服务器确认。换句话说，服务器1和2可能一个与旧Leader同步（即zxid与之相同）另一个不同步（即zxid比之小）。此时选票的更新主要取决于谁的zxid较大



在上图中，服务器1的zxid为11，而服务器2的zxid为10，因此服务器2将自身选票更新为（3, 1, 11），如下图所示。

![img](http://pcc.huitogo.club/b6586b69d641eb80ddb5eadf1a110423)



**3）选出新Leader**

经过上一步选票更新后，服务器1与服务器2均将选票投给服务器1，因此服务器2成为Follower，而服务器1成为新的Leader并维护与服务器2的心跳。



**4）旧Leader恢复后发起选举**

和Follower恢复后发起选举是一样的



#### 4. 集群读写流程

##### 4.1 读流程

Leader/Follower/Observer都可直接处理读请求，从本地内存中读取数据并返回给客户端

![img](http://pcc.huitogo.club/a023fc20556301fa0a2c2e0683547a29)



##### 4.2 写流程

对于Leader，写流程需要**同步给Follower和Observer节点**

![img](http://pcc.huitogo.club/65516d760322e6cb276d13bd82377a70)



这里需要注意的是

1. Leader并不需要得到Observer的ACK，即Observer无投票权
2. Leader不需要得到所有Follower的ACK，只要收到过半的ACK即可，同时Leader本身对自己有一个ACK。上图中有4个Follower，只需其中两个返回ACK即可，因为(2+1) / (4+1) > 1/2
3. Observer虽然无投票权，但仍须同步Leader的数据从而在处理读请求时可以返回尽可能新的数据



对于Observer或者Follower来说，写流程必须**转发给Leader**节点

![img](http://pcc.huitogo.club/bac36f9ffe6408cd462a31c357d023e9)



除了多了一步请求转发，其它流程与直接写Leader无任何区别

所以说**zk集群适合读多写少的场景**，而恰恰服务注册和发现就很符合这个特点。