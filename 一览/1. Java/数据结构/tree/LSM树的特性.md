在介绍LSM树之前，先来讲一下三种基本的存储引擎



- **哈希存储引擎**：是哈希表的持久化实现，支持增、删、改以及随机读取操作，但不支持顺序扫描，对应的存储系统为key-value存储系统。对于key-value的插入以及查询，哈希表的复杂度都是O(1)，明显比树的操作O(n)快,如果不需要有序的遍历数据，哈希表就是最佳选择。
- **B树存储引擎**：是B树的持久化实现，不仅支持单条记录的增、删、读、改操作，还支持顺序扫描（B+树的叶子节点之间的指针），对应的存储系统就是关系数据库（Mysql等）。
- **LSM树存储引擎**：和B树存储引擎一样，同样支持增、删、读、改、顺序扫描操作。而且通过批量存储技术规避磁盘随机写入问题。**当然凡事有利有弊，LSM树和B+树相比，LSM树牺牲了部分读性能，用来大幅提高写性能**。



**那么LSM树到底基于B树改变了什么？**

首先说一下传统的B树会有什么样的问题，比如mysql索引存储用B+树实现的，作为文件索引，当然经常和磁盘打交道了，我们知道CPU速度是内存的几百倍，而内存的运行速度是磁盘的几百万倍，所以经常跟磁盘打交道就意味着B树最大的**性能开销就是磁盘IO**了，虽然它已经尽力减少磁盘IO的次数。



比如查询元素或者插入节点，举一个插入key跨度很大的例子，如7->1000->3->2000 ... 新插入的数据存储在磁盘上相隔很远，会产生大量的随机读写IO。



那么LSM树的诞生是解决这个问题的吗？

**对头**。



为了更好的说明LSM树的原理，下面举个比较极端的例子：

现在假设有1000个节点的随机key，对于磁盘来说，肯定是把这1000个节点顺序写入磁盘最快，但是这样一来，读就悲剧了，因为key在磁盘中完全无序，每次读取都要全扫描；

那么，为了让读性能尽量高，数据在磁盘中必须得有序，这就是B+树的原理，但是写就悲剧了，因为会产生大量的随机IO，磁盘寻道速度跟不上。

LSM树本质上就是在读写之间取得平衡，和B+树相比，**它牺牲了部分读性能，用来大幅提高写性能**。



通过以上的分析，应该知道LSM树的由来了，LSM树的设计思想非常朴素：**将对数据的修改增量保持在内存中，达到指定的大小限制后将这些修改操作批量写入磁盘**，不过读取的时候稍微麻烦，需要合并磁盘中历史数据和内存中最近修改操作，所以写入性能大大提升，读取时可能需要先看是否命中内存，否则需要访问较多的磁盘文件。极端的说，基于LSM树实现的HBase的写性能比Mysql高了一个数量级，读性能低了一个数量级。



**LSM树原理把一棵大树拆分成N棵小树，它首先写入内存中，随着小树越来越大，内存中的小树会flush到磁盘中，磁盘中的树定期可以做merge操作，合并成一棵大树，以优化读性能。**



![img](http://pcc.huitogo.club/47181430b25cc1eb2300bb585558a9d0)



以上就是LSM树最本质的原理，来看一下LSM树的设计。



![img](http://pcc.huitogo.club/a3ab87f2703018ac94d3ad63f353669d)



**为什么要有WAL（Write Ahead Log）**

因为数据是先写到内存中，如果断电，内存中的数据会丢失，因此为了保护内存中的数据，需要在磁盘上先记录logfile，当内存中的数据flush到磁盘上时，就可以抛弃相应的Logfile。



**什么是memstore, storefile？**

上面说过，LSM树就是一堆小树，在内存中的小树即memstore，每次flush，内存中的memstore变成磁盘上一个新的storefile。



**为什么会有compact？**

随着小树越来越多，读的性能会越来越差，因此需要在适当的时候，对磁盘中的小树进行merge，多棵小树变成一颗大树。