我们先看一下消费者是怎么根据offsetId读取数据的？

如果我们要读取第 911 条数据。

- 首先第一步，找到它是属于哪一段的，根据二分法查找到他属于的文件，找到 `0000900.index` 和 `00000900.log` 之后。

- 然后，去 `.index` 中去查找 `(911-900) =11` 这个索引或者小于 11 最近的索引，在这里通过二分法我们找到了索引是 `[10,1367]` 。

  > - 10 表示，第 10 条消息开始。
  > - 1367 表示，在 `.log` 的第 1367 字节开始。

- 然后，我们通过这条索引的物理位置 1367 ，开始往后找，直到找到 911 条数据。



接下来看看Kafka的消息怎么存储的

#### 1. Kafka 存储在文件系统上

是的，您首先应该知道 **Kafka 的消息是存在于文件系统之上**的。**Kafka 高度依赖文件系统来存储和缓存消息**，一般的人认为 “磁盘是缓慢的”，所以对这样的设计持有怀疑态度。实际上，磁盘比人们预想的快很多也慢很多，这取决于它们如何被使用；一个好的磁盘结构设计可以使之跟网络速度一样快。



现代的操作系统针对磁盘的读写已经做了一些优化方案来加快磁盘的访问速度。比如，**预读**会提前将一个比较大的磁盘快读入内存。后写会将很多小的逻辑写操作合并起来组合成一个大的物理写操作。并且，操作系统还会将主内存剩余的所有空闲内存空间都用作**磁盘缓存**，所有的磁盘读写操作都会经过统一的磁盘缓存（除了直接 I/O 会绕过磁盘缓存）。综合这几点优化特点，**如果是针对磁盘的顺序访问，某些情况下它可能比随机的内存访问都要快，甚至可以和网络的速度相差无几**。



**上述的 Topic 其实是逻辑上的概念，面向消费者和生产者，物理上存储的其实是 Partition**，每一个 Partition 最终对应一个目录，里面存储所有的消息和索引文件。默认情况下，每一个 Topic 在创建时如果不指定 Partition 数量时只会创建 **1 个 Partition**。

![img](http://pcc.huitogo.club/4379ac3e8b6e8915bf1bba1e767385a4)



任何发布到 Partition 的消息都会被追加到 Partition 数据文件的尾部，这样的顺序写磁盘操作让 Kafka 的效率非常高（经验证，顺序写磁盘效率比随机写内存还要高，这是 Kafka 高吞吐率的一个很重要的保证）。

每一条消息被发送到 Broker 中，会根据 Partition 规则选择被存储到哪一个 Partition。如果 Partition 规则设置的合理，所有消息可以均匀分布到不同的 Partition中。



#### 2. Partition

在 Kafka 的文件存储中，同一个 Topic 下有多个不同的 Partition，每个 Partition 都为一个目录，而每一个目录又被平均分配成多个大小相等的 Segment File 中，Segment File 又由 index file 和 data file 组成，他们总是成对出现，后缀 “.index” 和 “.log” 分表表示 Segment 索引文件和数据文件。

Partition 中的每条 Message 都包含 3 个属性 Offset 、 MessageSize 、 Data。其中， Offset 表 示 Message 在这个 Partition 中的偏移盘，它在逻辑上是一个值，唯一确定了 Partition 中的 一 条 Message； MessageSize 表示消息内容 Data 的大小； Data 为 Message 的 具体内容。



Kafka Partition 中基于 Offset 的消息生产和消费如图所示

![img](http://pcc.huitogo.club/c94cc5c57543b22f9d7a03f3f1f6fd06)



#### 3. Segment

Partition 在物理上由多个 Segment 数据文件组成，每个 Segment 数据文件都大小相等 、 按顺序读写。每个 Segment 数据文件都以该段中最小的 Offset 命名，文件扩展名为.log。这样在查找指定 Offset 的 Message 的时候，用二分查找就可以定位到该 Message 在哪个 Segment 数据文件中 。

Segment 数据文件首先会被存储在内存中，当 Segment 上的消息条数达到配置值或消息发送时间超过阈值时，其上的消息会被 Flush 到磁盘，只有被 Flush 到磁盘的消息才能 被消费者消费到 。

Segment 达到 一定的大小(可以通过配置文件设定，默认为 1GB ) 后将不会再往该 Segment 中写数据， Broker 会创建新的 Segment。



现在假设我们设置每个 Segment 大小为 500 MB，并启动生产者向 topic1 中写入大量数据，topic1-0 文件夹中就会产生类似如下的一些文件：

![img](http://pcc.huitogo.club/b9807e51a33b171c1bc8d117fb038ef2)



**Segment 是 Kafka 文件存储的最小单位**。Segment 文件命名规则：Partition 全局的第一个 Segment 从 0 开始，后续每个 Segment 文件名为上一个 Segment 文件最后一条消息的 offset 值。数值最大为 64 位 long 大小，19 位数字字符长度，没有数字用0填充。如 00000000000000368769.index 和 00000000000000368769.log。



以上面的一对 Segment File 为例，说明一下索引文件和数据文件对应关系：

![img](http://pcc.huitogo.club/a226a03f3d69b61c0316de5e716edfdf)



其中以索引文件中元数据 <3, 497> 为例，依次在数据文件中表示第 3 个 message（在全局 Partition 表示第 368769 + 3 = 368772 个 message）以及该消息的物理偏移地址为 497。

注意该 index 文件并不是从0开始，也不是每次递增1的，这是因为 Kafka 采取稀疏索引存储的方式，每隔一定字节的数据建立一条索引，它减少了索引文件大小，使得能够把 index 映射到内存，降低了查询时的磁盘 IO 开销，同时也并没有给查询带来太多的时间消耗。

因为其文件名为上一个 Segment 最后一条消息的 offset ，所以当需要查找一个指定 offset 的 message 时，通过在所有 segment 的文件名中进行二分查找就能找到它归属的 segment ，再在其 index 文件中找到其对应到文件上的物理位置，就能拿出该 message 。



由于消息在 Partition 的 Segment 数据文件中是顺序读写的，且消息消费后不会删除（删除策略是针对过期的 Segment 文件），这种顺序磁盘 IO 存储设计师 Kafka 高性能很重要的原因。



#### 4. 数据文件索引

Kafka 为每个 Segment 数据文件都建立了索引文件以方便数据寻址，索引文件的文件 名与数据文件的文件名一致，不同的是索引文件的扩展名为 index。Kafka 的索引文件并 不会为数据文件中的每条 Message 都建立索引，而是采用**稀疏索引**的方式，每隔一定字节建立一 条奈'引 。 这样可以**有效地降低索引文件的大小**，方便将索引文件加载到内存中以提高集群的吞吐量。

> ps：RocketMQ中索引文件对每个数据文件中的消息，都有对应的索引。



索引文件中

- 第一位表示索引对应的 Message 的编号
- 第二位表示索引对应的 Message 的数据位置 。



Kafka lndex 的存储结构如图所示：

![img](http://pcc.huitogo.club/d13e80b081b2383686553327735c1253)

00000368769.index 索引文件采用稀疏索引的方式记录了第 1个 Message 、第 3 个 Message 、 第 6 个 Message 的索引分别为( 1, 0 )、 (3 ， 479)、( 6, 1407 )。



#### 5. message的物理结构

生产者发送到kafka的每条消息，都被kafka包装成了一个message



message 的物理结构如下图所示：

![](https://pcc.huitogo.club/kafka4.png)



所以生产者发送给kafka的消息并不是直接存储起来，而是经过kafka的包装，每条消息都是上图这个结构，只有最后一个字段才是真正生产者发送的消息数据。