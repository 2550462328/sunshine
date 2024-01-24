#### 1. 主从架构原理和缺陷

![img](http://pcc.huitogo.club/4a79dbb8c15c1e1eb87e5215775b0ccb)



master-slave架构中master节点负责数据的读写，slave没有写入权限只负责读取数据。

在主从结构中，主节点的操作记录成为oplog（operation log）。oplog存储在系统数据库local的oplog.$main集合中，这个集合的每个文档都代表主节点上执行的一个操作。从服务器会定期从主服务器中获取oplog记录，然后在本机上执行！对于存储oplog的集合，MongoDB采用的是固定集合，也就是说随着操作过多，新的操作会覆盖旧的操作。

主从结构没有自动故障转移功能，需要指定master和slave端，不推荐在生产中使用。

mongodb4.0后不再支持主从复制！



#### 2. 复制集

![img](http://pcc.huitogo.club/759ce06af235d7dbce0786c989c31a2e)



复制集是一个集群，它是2台及2台以上的服务器组成，以及复制集成员包括Primary主节点,secondary从节点和投票节点。一个复制集中Primary节点上能够完成读写操作,Secondary节点仅能用于读操作。

Primary节点需要记录所有改变数据库状态的操作,这些记录保存在 oplog 中,这个文件存储在 local 数据库,各个Secondary节点通过此 oplog 来复制数据并应用于本地,保持本地的数据与主节点的一致。

oplog具有幂等性,即无论执行几次其结果一致,这个比mysql的二进制日志更好用。



**心跳检测**

一个复制集N个节点中的任意两个节点维持心跳，每个节点维护其他N-1个节点的状态。

整个集群需要保持一定的通信才能知道哪些节点活着哪些节点挂掉。mongodb节点会向副本集中的其他节点每2秒就会发送一次pings包，如果其他节点在10秒钟之内没有返回就标示为不能访问。

每个节点内部都会维护一个状态映射表，表明当前每个节点是什么角色、日志时间戳等关键信息。如果主节点发现自己无法与大部分节点通讯则把自己降级为secondary只读节点。



**主节点选举触发的时机**

1. 第一次初始化一个复制集
2. Secondary节点权重比Primary节点高时，发起替换选举
3. Secondary节点发现集群中没有Primary时，发起选举
4. Primary节点不能访问到大部分(Majority)成员时主动降级



**选举过程**

当触发选举时,Secondary节点尝试将自身选举为Primary。主节点选举是一个二阶段过程+多数派协议。

第一阶段:

检测自身是否有被选举的资格（仲裁节点、投票节点不存储数据，无资格选举），如果符合资格会向其它节点发起本节点是否有选举资格的FreshnessCheck（询问其他节点关于“我”是否有被选举的资格）,进行同僚仲裁



检查项：

- 能ping通集群的过半数节点
- priority必须大于0
- 不能是arbitor节点



第二阶段:

发起者向集群中存活节点发送Elect(选举)请求，仲裁者收到请求的节点会执行一系列合法性检查，如果检查通过，则仲裁者(一个复制集中最多50个节点 其中只有7个具有投票权)给发起者投一票。

- pv0通过30秒选举锁防止一次选举中两次投票。
- pv1使用了terms(一个单调递增的选举计数器)来防止在一次选举中投两次票的情况。



**多数派协议**:

发起者如果获得超过半数的投票，则选举通过，自身成为Primary节点。获得低于半数选票的原因，除了常见的网络问题外，相同优先级的节点同时通过第一阶段的同僚仲裁并进入第二阶段也是一个原因。因此，当选票不足时，会sleep[0,1]秒内的随机时间，之后再次尝试选举



复制集搭建

![img](http://pcc.huitogo.club/eaef9308eb8185535acd5230d0e1c084)



#### 3. 分片集群 Shard Cluster

分片集群组成

- 分片节点 shard server
- 配置节点 config server
- 路由节点 rout server（mongos）

![img](http://pcc.huitogo.club/693baf07ab5d5198a33537fa4ac3d2a2)



分片集群由以下3个服务组成：

1. Shards Server: 每个shard由一个或多个mongod进程组成，用于存储数据。
2. Config Server: 配置服务器。存储所有数据库元信息（路由、分片）的配置。
3. Router Server: 数据库集群的请求入口，所有请求都通过Router(mongos)进行协调，不需要在应用程序添加一个路由选择器，Router(mongos)就是一个请求分发中心它负责把应用程序的请求转发到对应的Shard服务器上。



**片键（shard key）**

为了在数据集合中分配文档，MongoDB使用分片主键分割集合。



**区块（chunk）**

在一个shard server内部，MongoDB还是会把数据分为chunks，每个chunk代表这个shard server内部一部分数据。

每个chunk都是基于片键的范围取值，区间是左闭右开。

每个 chunk 位于哪个分片(Shard) 则记录在 Config Server(配置服务器)上。Mongos 在操作分片集合时，会自动根据分片键找到对应的 chunk，并向该 chunk 所在的分片发起操作请求。



**1）chunk分裂**

当某个chunk的值达到了chunk所能表示的最大值的时候（chunk默认大小为64MB），这个时候chunk不能无限增长，需要通过分割的方法来减少chunk的大小，例如一个64MB的chunk分割成2个32MB的chunk，这样虽然增加了chunk的数量，但是带来的收益是单个chunk的缩小。

![img](http://pcc.huitogo.club/a5e4561ddd63f99f2d2dc36c3bf1c6a0)



**2）chunk迁移**

在分片+复制集的架构中，当某个服务器上的数据记录不停的增多，它上面分割的chunk就会变多，当集群中每个服务器上的chunk数量严重失衡的时候，mongodb会自动进行chunk的迁移工作，这个自动迁移的工作，是通过balancer来进行的。如果balancer发现各个shard之间的chunk数差异超过了提前规定的阈值，则会进行chunk的迁移工作。

![img](http://pcc.huitogo.club/434248d81b9bde8189f3c14031530371)



**分片策略**

**1）范围分片（Range based sharding）**

![img](http://pcc.huitogo.club/b59871ed94c46defd9953fc206c1b7ba)



基于片键的值切分数据，每一个区块将会分配到一个范围。



优点：

适合满足在一定范围内的查找。

例如查找X的值在[20,30)之间的数据，mongo 路由根据Config server中存储的元数据，可以直接定位到指定的shard的Chunk中。



缺点：

如果shard key有明显递增（或者递减）趋势，则新插入的文档多会分布到同一个chunk，无法扩展写的能力。



**2）hash分片（Hash based sharding）**

![img](http://pcc.huitogo.club/7d4cf8c628d5c4b86649c0f893eb335c)



Hash分片是计算一个分片主键的hash值，每一个区块将分配一个范围的hash值。



优点：

能将文档随机的分散到各个chunk，充分的扩展写能力。



缺点：

不能高效的范围查询，所有的范围查询要分发到后端所有的Shard才能找出满足条件的文档。

一个理想的片键是同时拥有范围片键和哈希片键的优点，这点很难做到，关键是要理解自己的数据然后做出平衡。最好的效果就是**数据查询时能命中更少的分片，数据写入时能够随机的写入每个分片，关键在于如何权衡性能和负载**。