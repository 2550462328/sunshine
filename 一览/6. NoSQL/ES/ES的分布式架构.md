ElasticSearch 基于分布式的架构能够支撑 PB 级数据的搜索和分析。 ElasticSearch 分 布式架构的核心内容包括集群节点角色、集群选举原理、集群状态、数据路由规则、数 据分片和副本策略等。



#### 1. 集群搭建

##### 1.1 不同集群

不同的集群通过不同的名字来区分，默认名字“elasticsearch”

通过配置文件修改 或者在命令行中-E cluster.name=zlw进行设定

一个集群可以有一个或者多个节点



##### 1.2 同一集群下

每一个节点都有名字，通过配置文件配置，或者启动时候-E node.name=node1指定

每一个节点在启动之后，会分配一个UID，保存在data目录下

每个节点启动后，默认就是一个Master-eligible节点（可以设置node.master: false 禁止）

Master-eligible节点可以参加选主流程，成为Master节点；当第一个节点启动时候，它会将自己选举成Master节点；

每个节点上都保存了集群的状态，只有Master节点才能修改集群的状态信息，包括

- 所有的节点信息
- 所有的索引和其相关的Mapping与Setting 信息
- 分片的路由信息



节点分为Data Node 和Coordinating Node

- Data Node：可以保存数据的节点，负责保存分片数据。在数据扩展上起到了至关重要的作用。
- Coordinating Node：负责接受Client的请求，将请求分发到合适的节点，最终把结果汇集到一起（每个节 点默认都起到了Coordinating Node的职责）



#### 2. 分片和副本

##### 2.1 分片和副本概念

- **主分片**，用以解决数据水平扩展的问题。通过主分片，可以将数据分布到集群内的所有节点之上

一个分片是一个运行的Lucene的实例

主分片数在索引创建时指定，后续不允许修改，除非Reindex



- **副本**，用以解决数据高可用的问题。分片是主分片的拷贝

副本分片数，可以动态的调整

增加副本数，还可以在一定程度上提高服务的可用性(读取的吞吐)



下面是在一个三节点的集群中，blogs 索引的分片分布情况

![img](http://pcc.huitogo.club/cb2f7ec1b1a0d76585e38cbf0d2a9f5d)

![img](http://pcc.huitogo.club/afe28d18f61b49a0a4f50af53cc5b914)



##### 2.2 对于分片的设定

对于生产环境中分片的设定，需要提前做好容量规划.

- 分片数设置过小

导致后续无法增加节点实现水平扩展

单个分片的数据量太大，导致数据重新分配耗时



- 分片数设置过大，7.0开始，默认主分片设置成1，解决了over-sharding的问题

影响搜索结果的相关性打分，影响统计结果的准确性

单个节点上过多的分片，会导致资源浪费,同时也会影响性能



#### 3. 集群节点角色

ElasticSearch 集群节点角色包括 MasterNode (主节点)、 DataNode (数据节点)、 IngestNode (提取节点)、 CoordinatingNode (协调节点)和 TribeNode (部落节点)。



ElasticSearch 集群各角色及其关系如图所示

![img](http://pcc.huitogo.club/478794f0522ae1bf90b7f814f71f0362)



1）**MasterNode** (主节点)：MasterNode 主要负责集群节点状态的维护、索引的创建 删除 、数据 的 Rebalance 、分片的分配等工作 。 MasterMode 不负责具体数据的索引和检索， 因此其负载较低，服务比较稳定。当 MasterNode 宕机时，ElasticSearch 集群会自动从其 他 MasterNode 中选举出一个 Leader 继续为集群提供服务 。为了防止在选举过程中出现脑 裂现象，常常简要设置 discovcry.zcn.minimum_master_nodes=N/2+ 1，其中 N 为集群中 MasterNode 的个数 。建议集群中 MasterNode 的个数为奇数，如 3 个或者 5 个。



一个节点 只包含 MasterNode 角色的配置如下。

```
node.master: true #设置节点角色为 MasterNode

node.data: false #设置节点角色为非 DataNode
node.ingest: false #设置节点角色为非 IngestNode

search.remote.connect: false #设置节点角色为非查询角色
```

在一般生产环境中 ，为了保障 MasterNode 的稳定运行，不建议在 MasterNode 上配 置数据节点。



2） **DataNode** (数据节点)：DataNode 是集群的数据节点，主要负责集群中数据的 索引创建和检索 ，具体操作包括数据的索引 、搜索、聚合等。 DataNode 属于I/O 、内存 和 CPU 密集型操作，需要的计算资源较大 ，如果资源允许，则建议使用 SSD 以加快数据 读写的效率。



设置一个节点为 DataNode 的配置如下

```
node.master: false
node.data: truet设置节点角色为 DataNode
node . ingest : false
search.remote.connect : false
```



3）**IngestNode** (提取节点) ：ngestNode 是执行数据预处理的管道，它在索引之前 预处理文档。通过拦截文档的 Bulk 和 lndex 请求 然后加 以转换，最终将文档传回 Bulk 和 lndex API ，用户可以定义一个管道，指定一系列预处理器。如果集群有复杂的数据预 处理逻辑 ，则该节点属于高负载节点，建议使用专用服务器。



设置一个节点为 IngestNode 的配置如下 ：

```
node.maste: false
node.data: false
node.ingest: true
cluster.rernote.connect: false
```



4）**CoordinatingNode** (协调节点)：CoordinatingNode 用于接收客户端请求，并将请 求转发到各个 DataNode 上。各个 DataNode 在收到请求后，在本地执行请求操作，并将 请求结果反馈给 CoordinatingNode ， CoordinatingNode 在收到所有 DataNode 的反馈后， 进行结果合并 ， 然后将结果返回客户端。



设置一个节点为 CoordinatingNode 的配置如下 。

```
node.master: false
node.data: false
node.ingest: false
```



5）**TribeNode** (部落节点)：允许 TribeNode 在多个集群之间充当联合客户端，用于 实现跨集群访问。在 5 .4.0 版本以后 ，TribeNode 已经被废弃，并不建议使用 ， 其替代方 案为 cross-cluster Search 。



#### 4. 节点选举流程

ElasticSearch 集群选主操作采用**Bully算法**实现 。

Bully 算法是 Leader 选举的基本算法之一 ， 其优点是易于实现。该算法假定所有节点都有一个唯一的 id ，使用该 id 对节点进行排序，每次都会选出存活的进程中 id 最大的节点作为候选者。需要注意的是， ElasticSearch在选举算法的具体实现过程中，其选举实现类 ElectMasterService 将该算法的实现进行了修改，选用最小id作为候选者 。



选举过程如下：

1. 对所有可以成为master的节点根据nodeId排序，每次选举每个节点都把自己所知道节点排一次序，然后选出第一个（第0位）节点，暂且认为它是master节点。
2. 如果对某个节点的投票数达到一定的值（可以成为master节点数n/2+1）并且该节点自己也选举自己，那这个节点就是master，开始发布集群状态 。否则重新选举。
3. ElasticSearch 中的所有节点都会参与选举和投票， 但只有拥有 Master 角色的节点 投票才有效。
4. 对于brain split问题，需要把候选master节点最小值设置为可以成为master节点数n/2+1



首先**什么是脑裂问题？**

在集群环境下，当出现网络问题，一个节点和其他节点无法连接

Node 2 和 Node 3 会重新选举 Master

Node 1 自己还是作为 Master，组成一个集群，同时更新 Cluster State

导致 2 个 master，维护不同的 cluster state。当网络恢复时，无法选择正确的集群状态。



**Elasticsearch怎么解决脑裂问题的？**

ElasticSearch 通过 discovery.zenminimum_master_nodes 来控制选主的条件。

```
discovery.zen.minimum_master_nodes= (master_eligible_nodes) /2+1
```



在上述配置中， ( master_eligibl e_nodes) /2+ 1 表示大于半数节点个数， ElasticSearch 基于该配置在集群选主的时候会做如下判断

1. 触发选主： 在进入选举临时 Master 之前，参选的节点数需要达到法定数量 。
2. 决定 Master：在选 出 临时 Master 之后，得票数需要达到法定数量，才确认选主成 功。
3. Gateway 选举元信息：向有 Master 资格的节点发起请求，获取元数据，获取的响应数量必须达到法定数量，也就是参与元信息选举的节点数 。
4. Master 发布集群状态：成功向节点发布集群状态信息的数量要达到法定数量 。
5. NodesFaultDetection事件中是否触发rejoin ：当发现有节点连不上时，会执行removeNode。接着审视此时的法定数量是否大于discovery.zen.minimum_master_ nodes 配置，如果不大于，则主动放弃 Master 身份执行 rejoin 以避免脑裂 。



比方说假设集群中有5个节点且5个节点都可以成为Master节点，因此minimumMasterNodes的数目为3（3=5/2+1）。

假设之前选举了A节点为master，两个switch之间突然断线了，这样就分成了两部分。CDE和AB，因为 minimumMasterNodes的数目为3，因此cde会可以进行选举假设C成为master。AB两个节点因为少于3所以无法选举，只能一直寻求加入集群，要么线路连通加入到CDE中要么就一直处于寻找集群状态，这样就保证了集群不分裂。



#### 5. 集群状态

ElasticSearch 集群的状态分为 Gree、Yellow 和 Red 3 种 。



ElasticSearch 集群的状态 及其关系如图所示。

![img](http://pcc.huitogo.club/79cb508f377856e358b2fb4b967ea621)



- Green：所有主分片和副本分片都运行正常 。
- Yellow：所有主分片都运行正常，但至少还有一个副本分片运行异常 。 在这种 状态下，集群仍然可以对外提供服务，并且可以保障数据的完整性，但是集群的高可用 不如 Green。当集群状态为 Yellow 时，如果有一个主分片的节点出现故障，则集群将有 数据缺失的风险。 Yellow 可以理解为需要进行故障维护的状态 。
- Red：至少有一个主分片运行异常 。 这时集群的搜索请求只能返回部分数据，而 分配到这个分片上的写入请求将会返回异常 。



#### 6. 数据路由规则

ElasticSearch 的数据路由( Routing )规则用于确定文档存储在哪个索引( lndex )的哪个分片( Shard )上。根据路由规则， ElasticSearch 将不同文档索引到不同索引的不同分片上 。 在查询文档的时候 ，ElasticSearch 根据路由规则找到该索引及其对应的分片并查询该文挡 。



其路由规则的公式如下。

```
shard= hash(routing)%number_of_primary_shards
```



在上述公式中， routing 是一个可变值， 默认是文档的id ，也可以设置成一个自定义 的值 。 上述公式简述为文档所在分片等于 routing 的 Hash 值除以主分片数 ( number_of_p rimary_shards ) 的余数。这也是为什么 ElasticSearch 索引的主分片数量在确定后就不能再修改的原因，因为如果主分片数量发生变化，则之前路由的所有分片都会失效 。

在使用时，所有 API ( get 、 index 、 delete 、 bulk 、 update 以及 mget) 都接收一个叫作 routing 的路由参数，通过这个参数应用程序可以自定义文档到分片的映射 。一 个自定义的路由参数可以用来确保所有相关的文档（例如所有属于同 一个用户的文档）都被存储到同一个分片中。



#### 7. 文档分片和副本策略

ElasticSearch 文档分片的原则如下 。

1. ElasticSearch 中的每个索引都由 一个或多个分片组成，文档根据路由规则分配到 不同分片 上。
2. 每个分片都对应一个 Lucene 实例， 一个分片只能在放 Integer. MAX_VALUE - 128 = 2147483519 个文档 。
3. 分片主要用于数据的横向分布， ElasticSearch 中的分片会被尽可能平均地分配到 不同节点上 ， 当有新的节点加入时 ， ElasticSearch 会自动感知并对数掘进行 relocation 操 作(例如，有 2 个节点， 4 个主分片，那么每个节点都将会分到 2 个分片，当再增加 2 个 节点后， ElasticSearch 会自动执行 relocation 操作，这时每个节点都将会分到1个分片)， relocation 保 障了集群内数据的均衡分布 。



ElasticSearch 文档副本的策略如下 。

1. ElasticSearch 的副本即主分片 ( Primary Shard) 对应数据的副本分片( Replica Shard )。
2. 为了防止单节点服务器故障， ElasticSearch 会将主分片和副本分片分配在不同节点上 。 ElasticSearch 的默认配置是一个索引包含 5 个分片，每个分片都有 l 个副本(即 5 Primary+5 Replica=10 个分片)。



下图表示一个有 3 个节点的 ElasticSearch 集群 ， 该集群中有一个名称为 lndex-1 的 索引，该索引被分成 2 个 Shard ， 分别为 Shard-0 和 Shard-1。 2 个 Shard 对应的主分片分

别为 P-0 和 P-1 ，副本分片分别为 R-0 和 R-1 ，各个分片都均匀地分布在 3 个节点上。其 中， Node-1 上分配的分片为 P-0 和 R-1 ， Node-2 上分配的分片为R-0和 R-1， Node-3 上 分配的分片为 R-0 和 P-1 。 当 Node-1 服务若机时， Shard-0 的主分片 P-0 将不可用，这时 ElasticSearch 会将 Node-2 或 Node-3 上的副本分片 R-0 升级为主分片对外提供服务，以实现高可用 。

![img](http://pcc.huitogo.club/4b5ae0f58ff745042f79788fe94374e7)