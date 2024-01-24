#### 1. JVM 性能调优

ElasticSearch 基于 Java 实现，默认使用的堆内存为 1GB ，对于生产环境需要根据系 统资源对堆内存进行合理的设置以达到良好的性能表现 。



执行以下命令对 JVM 堆内存进 行设置。

```
vim config/jvm.options
```



如果操作系统有 32GB 内存，则建议将 JVM 堆内存的最小值和最大值都设置为 16GB。

```
-Xms16gb
-Xmx16gb
```



这里将堆内存最小值 (Xms) 与最大值 (Xmx) 设置相间，防止在 ElasticSearch 运 行过程中 JVM 改变堆 内存大小，引起 JVM 内存震荡 。

需要注意的是， ElasticScarch 除了使用 JVM 堆内存，其内部 Lucene 还需要使用大量 非堆内存。 ElasticSearch 内部使用 Lucene 实现全文检索。 Lucene 的段分别存储在单个文 件中，因为段是不可变的，对缓存友好的，所以在使用段数据时操作系统会把这些段文 件缓存起来，以便更快地访问。 同时， Lucene 可以利用操作系统底层机制来缓存内存数 据，加速查询效率 。

Lucene 的性能取决于与操作系统交互的速度，而这些交互都需要大量的内存资源 (非 JVM 堆内存) ， 如果把全部内存都分配给 JVM 堆内存，则将导致 Lucene 在运行过程 中 因资源不足而性能下降 。一般建议将系统的 一半 内存分配给 JVM 堆内存，另外一半内 存预留给 Lucene 和操作系统 。 比如有 32GB 内存。 可以把 16GB 分配给 JVM 堆内存，剩 余的 16GB 预留给 Lucene 和操作系统 。



#### 2. 操作系统的性能调优

1）**设置文件句柄**：Linux 中的每个进程默认打开的最大文件句柄数都是 1024 ，对 于服务器进程来说该值太小 ， 可以通过修改 /etc/security/ Iimits.conf 来增大打开的最大文件句柄数 ， 一般建议设置为 65535 。设置命令如下。

```
echo "* soft nofile 65535" >> /etc/security/limits.conf
echo "* hard nofile 65535" >> /etc/security/limits.conf
```



2 ）**设置虚拟内存**：max_map_count 定义 了进程能拥有的最多内存区域， 一般建议 设置为 102 400。设置命令如下。

```
sysctl -w vm.max_map_count=102400
```



3 ）**关闭 Swap**：Swap 空间是一块磁盘空间， 操作系统使用这块空间保存从内存中 交互换出的操作系统不常用的 Page 数据 ， 这样可以分配出更多的内存做 Page Cache。通 过 Swap 可以提升系统的吞吐量和 I/O性能 ， 但 ElasticSearch 需要一个所有内存操作都能 够被快速执行的环境， 服务一旦使用到 了 Swap内存 ， 就会大大降低数据的存取效率，严 重影响性能 。 关闭设置的命令如下。

```
echo "vrn.swappiness=0" >> /etc/sysctl.conf
```



4 ）**开启 mlockall**：打开配置文件中的 mlockall 开关。它的作用是允许 JVM 锁住内存，禁止操作系统将内存交换出去。 elasticsearch. yml文件中的设置如下。

```
bootstrap.mlockall: true
```