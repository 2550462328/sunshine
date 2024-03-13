#### 1. ES 写数据过程

ElasticSearch 的写操作主要包括索引的创建和删除，以及文档的创建、删除、更新等操作 。 ElasticSearch 首先会在主分片上执行写操作，当主分片上执行成功时，根据集群的 数据一致性要求，将在其他副本分片上执行写操作，只有达到一致性要求的节点都执行成功后才向客户端发送成功响应 。

当向 ElasticSearch 发送请求时，可以将请求发送到集群中的任一节点 。 ElasticSearch 集群中的每个节点都有能力处理任意请求。 每个节点都知道集群中任一文档的位置。接 收请求的节点被称为协调节点 。



ElasticSearch文档的写入流程如图所示：

![es-write](https://pcc.huitogo.club/z0/es-write.png)

1. 客户端选择一个 node 发送请求过去，这个 node 就是 `coordinating node` （协调节点）。

2. `coordinating node` 对 document 进行**路由**，将请求转发给对应的 node（有 primary shard）。

3. 实际的 node 上的 `primary shard` 处理请求，然后将数据同步到 `replica node` 。

4. `coordinating node` 如果发现 `primary node` 和所有 `replica node` 都搞定之后，就返回响应结果给客户端。

  

**怎么保证写入操作的数据一致性？**

ElasticSearch 通过 Consistency 设置写操作的 一致性级别 。 Consistency 的参数值可以 是 one (仅主要分片)、 all (主分片和所有副本分片)，或者是默认的 Quorurn (大部分分 片)。

在默认情况下，主分片需要通过 Quorurn，即确认大部分副本分片有效时，才会发起 一个写操作 。这样做的目的是防止将数据写入网络中"错误的一侧 (Wrong Side)" 。



Quorurn 的定义如下

```
quorum = int ((primary+number_of_replicas) /2) +1
```



上述公式的 number_of_replicas 指定在索引设置中的副本分片的数量，而不是当前处于活动状态的副本分片数量 。 例如，在索引中指定了有 1个主分片、 3 个副本分片，那么 Quorum 的计算结果为 3。



#### 2. ES数据安全

ES写入磁盘的数据流程：

![es-write-detail](https://pcc.huitogo.club/z0/es-write-detail.png)

**总结：ES是近实时搜索的，延迟为1s；可能会丢失5s内的数据**

1. 数据写入primary shard节点的内存buffer区 和 translog中
2. 每隔1s将内存buffer区的数据 refresh 到 os cache中 且 清空内存buffer，这个时候写入的数据已经可以被搜索到
3. os cache的数据会被fsync刷盘刷到 segment file的磁盘文件中，所以理论上如果一直有数据写入的话 ES会每秒生成一个segment file
4. 当translog达到一定长度后会 ES会触发commit操作，这个commit操作需要确保内存中的所有数据都被已经被写到磁盘中，同时需要记录写入磁盘的segment file
5. segment file达到一定数量后会执行merge操作，将多个segment file合成一个 并且 物理删除掉 被标记为删除的文档，最后将合并后的segment file写入磁盘提供搜索。



##### 2.1 什么是Refresh?、

将 Index buffer 写入 Segment 的过程叫 Refresh。Refresh 不执行 fsync 操作

![img](http://pcc.huitogo.club/adf5fac67c03f36b5fb67673d93e7a7e)



Refresh 频率：默认 1 秒发生一次，可通过 index.refresh_interval 配置。Refresh 后， 数据就可以被搜索到了。这也是为什么 Elasticsearch 被称为近实时搜索。

如果系统有大量的数据写入，那就会产生很多的 Segment

只要数据被输入 `os cache` 中，buffer 就会被清空了，因为不需要保留 buffer 了，数据在 translog 里面已经持久化到磁盘去一份了。

Index Buffer 被占满时，会触发 Refresh，默认值是 JVM 的 10%



##### 2.2 什么是Flush？

commit 操作发生第一步，就是将 buffer 中现有数据 `refresh` 到 `os cache` 中去，清空 buffer。然后，将一个 `commit point` 写入磁盘文件，里面标识着这个 `commit point` 对应的所有 `segment file` ，同时强行将 `os cache` 中目前所有的数据都 `fsync` 到磁盘文件中去。最后**清空** 现有 translog 日志文件，重启一个 translog，此时 commit 操作完成。

这个 commit 操作叫做 `flush` 。默认 30 分钟自动执行一次 `flush` ，但如果 translog 过大（默认 512 MB），也会触发 `flush` 。flush 操作就对应着 commit 的全过程，我们可以通过 es api，手动执行 flush 操作，手动将 os cache 中的数据 fsync 强刷到磁盘上去。



##### 2.3 translog 日志文件的作用是什么？

**1）什么是ElaslicSearch Translog？**

ElasticSearch 的 事务 日志文件为 Translog，记录了所有与更新相关的事务操作日志 (例如 add/update/delete)。在 ElasticSearch 中 ，索引被分为多个分片 ， 每个分片都对应一 个 Translog 文件。 Translog 提供了数据没有被刷盘的一个持久化存储纪录，主要用来恢 复数据。当 ElasticSearch 重启时，它会在磁盘中使用最后一个提交点( Commit Point )去 恢复已知的段数据，并且会重放 Translog 中所有在最后一次提交后发生的变更操作 。



**2）ElasticSearch Translog在更新操作中怎么使用的？**

Translog 被用来提供实时 CRUD 。当我们试着通过 id 查询、更新 、 删除一个文档时， ElasticSearch 会在尝试从相应段中检索之前，首先检查 Translog 的最近变更记录，以保障客户端查询总是能够实时地获取文档的最新版本。



下面的参数可以用来设置 Translog 的刷新策略 。

1）`index.translog.sync_interval`：每间隔多久事务日志执行 fsync 操作和提交写操作 。 默认为5s，设置的值需要大于等于 100ms。

2）`index.translog.durability`：是否在每次 index 、 delete 、 update 、 bulk 请求后都去执 行 fsync 和提交事务日志的操作，具体参数设置如下 。

- request : 每次请求均执行 fsync 和提交操作 。 它可保障发生硬件故障时，所有确认的写操作都将被写入磁盘 。

- sync: 每隔 sync_interval 时间将会执行 fsync 和提交操作。 当发生硬件故障时， 所 有确认的写操作在上次自动提交后都会被丢弃 。

3）`index.translog.flush_threshold_size`：当事务日志中存储的数据大于该值时，则启 动刷新操作，产生一个新的 Commit Point。 默认值为 512MB。

4）`index.translog.retention.size`：保留事务日志的总大小。当恢复备份数据时，太多 事务日志文件将增大基于 sync 操作的概率；如果事务日志不够，则备份恢复将会回退到 基于 sync 的文件。 默认值为 512MB，在 ElasticSearch 7.0 . 0 及更高版本中，该配置已经 被废弃 。

5）`index.translog. retention.age`：事务日志保留的最长时间，默认为12h，在 ElasticSearch 7.0.0 及更高版本中，该配置已经被废弃 。



##### 2.4 为什么会丢失5s内的数据

translog 其实也是先写入 os cache 的，默认每隔 5 秒刷一次到磁盘中去，所以默认情况下，可能有 5 秒的数据会仅仅停留在 buffer 或者 translog 文件的 os cache 中，如果此时机器挂了，会**丢失** 5 秒钟的数据。但是这样性能比较好，最多丢 5 秒的数据。也可以将 translog 设置成每次写操作必须是直接 `fsync` 到磁盘，但是性能会差很多。

- `index.translog.sync_interval` 控制 translog 多久 fsync 到磁盘,最小为 100ms；
- `index.translog.durability` translog 是每 5 秒钟刷新一次还是每次请求都 fsync，这个参数有 2 个取值：request(每次请求都执行 fsync,es 要等 translog fsync 到磁盘后才会返回成功)和 async(默认值，translog 每隔 5 秒钟 fsync 一次)。



##### 2.5 什么是段合并

Lucene 索引是由多个段组成，段本身是一个功能齐全的倒排索引。

由于自动刷新流程每秒都会创建一个新的段，这样会导致短时间内段的数量迅速增 加。 而段的数量太多将会引起性能瓶颈 。 在 ElasticSearch 中，每一个段都会消耗文件句 柄 、 内存和 CPU 运行周期 。 同时，每个搜索请求都必须轮流检查每个段；因此段越多 ， 搜索也就越慢 。 ElasticSearch 通过在后台进行段合并来解决这个问题 。 小的段被合并到大的段，大的段再被合并到更大的段。

在段合并的时候会将那些旧的已删除文档从文件系统中消除 。 被删除的文档(或被 更新文档的旧版本)不会被复制到新的大段中。段合并在进行索引和搜索时会自动进行 。



**段合并的流程**

将两个提交了的段和一个未提交的段合并到一个更大的段，其流程如图所示

![img](http://pcc.huitogo.club/3ab5f0afca6b79ef846d69905e5f5c43)



1. 在索引的时候， Refresh 操作会创建新的段并将段打开以供搜索使用 。
2. 合并进程选择一小部分大小相似的段，并且在后台将它们合并到更大的段中 。 该操作并不会中断索引和搜索 。
3. 在合并结束后，老的段被删除，新的段被 Flush 到磁盘 ( 写入一个包含新段且排 除旧段和较小段的新提交点)，再打开新的段对外提供搜索，删除老的段数据。

合并大的段需要消耗大量的 I/O 和 CPU 资源， ElasticSearch 对合并流程进行资源限制，以保障搜索仍然有足够的资源被正常地执行 。



可以通过 API 设置 max_bytes_per_sec 来提高段合并的性能，默认值为 20MB/s ，对 于机械磁盘， 20MB/s 是合理的设置，但如果磁盘是 SSD ， 则考虑提高到 100~200MB/s ，设置 API 如下 。

```
PUT /_cluster/settings
{
  "persistent": {
    "indices.store.throttle.max_bytes_per_sec": "100mb"
  }
}
```



可以通过以下 API 彻底关掉段合并限流，使段合并速度尽可能快，上限为磁盘允许 的最大读写速度。

```
PUT /cluster/settings
{
  "transient": {
    "indices.store.throttle.type": "none"
  }
} 
```



同时，可以在 elasticsearch.yml 配置中设置 max-thread-count 的个数 以提高段合并的 并发度。

```
index.merge.scheduler.max_thread_count: 10
```



#### 3. ES删除数据

删除操作，commit 的时候会生成一个 `.del` 文件，里面将某个 doc 标识为 `deleted` 状态，那么搜索的时候根据 `.del` 文件就知道这个 doc 是否被删除了。



#### 4. ES修改数据

更新操作，就是将原来的 doc 标识为 `deleted` 状态，然后新写入一条数据。

buffer 每 refresh 一次，就会产生一个 `segment file` ，所以默认情况下是 1 秒钟一个 `segment file` ，这样下来 `segment file` 会越来越多，此时会定期执行 merge。每次 merge 的时候，会将多个 `segment file` 合并成一个，同时这里会将标识为 `deleted` 的 doc 给**物理删除掉**，然后将新的 `segment file` 写入磁盘，这里会写一个 `commit point` ，标识所有新的 `segment file` ，然后打开 `segment file` 供搜索使用，同时删除旧的 `segment file` 。

