#### 1. 消息存储整体架构

消息存储架构图中主要有下面三个跟消息存储相关的文件构成。

- #####  CommitLog

  消息主体以及元数据的存储主体，存储Producer端写入的消息主体内容,消息内容不是定长的。单个文件大小默认1G ，文件名长度为20位，左边补零，剩余为起始偏移量，比如00000000000000000000代表了第一个文件，起始偏移量为0，文件大小为1G=1073741824；当第一个文件写满了，第二个文件为00000000001073741824，起始偏移量为1073741824，以此类推。消息主要是顺序写入日志文件，当文件满了，写入下一个文件；

- #####  ConsumeQueue

  消息消费队列，引入的目的主要是提高消息消费的性能，由于RocketMQ是基于主题topic的订阅模式，消息消费是针对主题进行的，如果要遍历commitlog文件中根据topic检索消息是非常低效的。Consumer即可根据ConsumeQueue来查找待消费的消息。其中，ConsumeQueue（逻辑消费队列）作为消费消息的索引，保存了指定Topic下的队列消息在CommitLog中的起始物理偏移量offset，消息大小size和消息Tag的HashCode值。consumequeue文件可以看成是基于topic的commitlog索引文件，故consumequeue文件夹的组织方式如下：topic/queue/file三层组织结构，具体存储路径为：$HOME/store/consumequeue/{topic}/{queueId}/{fileName}。同样consumequeue文件采取定长设计，每一个条目共20个字节，分别为8字节的commitlog物理偏移量、4字节的消息长度、8字节tag hashcode，单个文件由30W个条目组成，可以像数组一样随机访问每一个条目，每个ConsumeQueue文件大小约5.72M；

- ##### IndexFile

  IndexFile（索引文件）提供了一种可以通过key或时间区间来查询消息的方法。

  Index文件的存储位置是：$HOME \store\index${fileName}，文件名fileName是以创建时的时间戳命名的，固定的单个IndexFile文件大小约为400M，一个IndexFile可以保存 2000W个索引，IndexFile的底层存储设计为在文件系统中实现HashMap结构，故rocketmq的索引文件其底层实现为hash索引。



总结来说，整个消息存储的结构，最主要的就是 CommitLoq 和 ConsumeQueue 。而 ConsumeQueue 你可以大概理解为 Topic 中的队列。

![img](http://pcc.huitogo.club/03bbf46be94e1a23b097c29241b73197)



下图是官方的消息存储整体架构：

![img](http://pcc.huitogo.club/3b9e1a22796eb140c8090704fa94bdee)



上面图中假设Consumer端默认设置的是同一个ConsumerGroup，因此Consumer端线程采用的是负载订阅的方式进行消费。从架构图中可以总结出如下几个关键点：

- **消息生产与消息消费相互分离**，Producer端发送消息最终写入的是CommitLog（消息存储的日志数据文件），Consumer端先从ConsumeQueue（消息逻辑队列）读取持久化消息的起始物理位置偏移量offset、大小size和消息Tag的HashCode值，随后再从CommitLog中进行读取待拉取消费消息的真正实体内容部分；
- **RocketMQ的CommitLog文件采用混合型存储**（所有的Topic下的消息队列共用同一个CommitLog的日志数据文件），并通过建立类似索引文件—ConsumeQueue的方式来区分不同Topic下面的不同MessageQueue的消息，同时为消费消息起到一定的缓冲作用（只有ReputMessageService异步服务线程通过doDispatch异步生成了ConsumeQueue队列的元素后，Consumer端才能进行消费）。这样，只要消息写入并刷盘至CommitLog文件后，消息就不会丢失，即使ConsumeQueue中的数据丢失，也可以通过CommitLog来恢复，正因为如此，Consumer也就肯定有机会去消费这条消息。当无法拉取到消息后，可以等下一次消息拉取，同时服务端也支持长轮询模式，如果一个消息拉取请求未拉取到消息，Broker允许等待30s的时间，只要这段时间内有新消息到达，将直接返回给消费端。
- **RocketMQ每次读写文件的时候真的是完全顺序读写么？**这里，发送消息时，生产者端的消息确实是**顺序写入CommitLog**；订阅消息时，消费者端也是**顺序读取ConsumeQueue**，然而根据其中的起始物理位置偏移量offset读取消息真实内容却是**随机读取CommitLog**。 在RocketMQ集群整体的吞吐量、并发量非常高的情况下，随机读取文件带来的性能开销影响还是比较大的，那么这里如何去优化和避免这个问题呢？



这里，同样也可以总结下RocketMQ存储架构的优缺点：

- **优点：**

   a、ConsumeQueue消息逻辑队列较为轻量级；

   b、对磁盘的访问串行化，避免磁盘竟争，不会因为队列增加导致IOWAIT增高；

- **缺点：**

   a、对于CommitLog来说写入消息虽然是顺序写，但是读却变成了完全的随机读；

   b、Consumer端订阅消费一条消息，需要先读ConsumeQueue，再读Commit Log，一定程度上增加了开销；



#### 2. 存储关键技术 - 页缓存与内存映射

##### 2.1 Mmap内存映射技术—MappedByteBuffer

Mmap内存映射和普通标准IO操作的本质区别在于它并不需要将文件中的数据先拷贝至OS的内核IO缓冲区，而是可以直接将用户进程私有地址空间中的一块区域与文件对象建立映射关系，这样程序就好像可以直接从内存中完成对文件读/写操作一样。只有当缺页中断发生时，直接将文件从磁盘拷贝至用户态的进程空间内，只进行了一次数据拷贝。对于容量较大的文件来说（文件大小一般需要限制在1.5~2G以下），采用Mmap的方式其读/写的效率和性能都非常高。

![img](https://pcc.huitogo.club/z0/4325076-76cd8768f717a2bc.jpg)

> 注：正因为需要使用内存映射机制，故RocketMQ的文件存储都使用定长结构来存储，方便一次将整个文件映射至内存。



***Q1：MappedByteBuffer咋实现的？***

从JDK的源码来看，MappedByteBuffer继承自ByteBuffer，其内部维护了一个逻辑地址变量—address。在建立映射关系时，MappedByteBuffer利用了JDK NIO的FileChannel类提供的map()方法把文件对象映射到虚拟内存。仔细看源码中map()方法的实现，可以发现最终其通过调用native方法map0()完成文件对象的映射工作，同时使用Util.newMappedByteBuffer()方法初始化MappedByteBuffer实例，但最终返回的是DirectByteBuffer的实例。在Java程序中使用MappedByteBuffer的get()方法来获取内存数据是最终通过DirectByteBuffer.get()方法实现（底层通过unsafe.getByte()方法，以“地址 + 偏移量”的方式获取指定映射至内存中的数据）。



***Q2：Mmap有什么限制？***

- **Mmap映射的内存空间释放的问题**；由于映射的内存空间本身就不属于JVM的堆内存区（Java Heap），因此其不受JVM GC的控制，卸载这部分内存空间需要通过系统调用 unmap()方法来实现。然而unmap()方法是FileChannelImpl类里实现的私有方法，无法直接显示调用。**RocketMQ中的做法是**，通过Java反射的方式调用“sun.misc”包下的Cleaner类的clean()方法来释放映射占用的内存空间；
-  **MappedByteBuffer内存映射大小限制**；因为其占用的是虚拟内存（非JVM的堆内存），大小不受JVM的-Xmx参数限制，但其大小也受到OS虚拟内存大小的限制。一般来说，一次只能映射1.5~2G 的文件至用户态的虚拟内存空间，这也是为何RocketMQ默认设置单个CommitLog日志数据文件为1G的原因了；
-  **使用MappedByteBuffe的其他问题**；会存在内存占用率较高和文件关闭不确定性的问题；



##### 2.2 OS的PageCache机制

PageCache是OS对文件的缓存，用于加速对文件的读写。一般来说，程序对文件进行顺序读写的速度几乎接近于内存的读写访问，这里的主要原因就是在于OS使用PageCache机制对读写访问操作进行了性能优化，将一部分的内存用作PageCache。

1. **对于数据文件的读取**，如果一次读取文件时出现未命中PageCache的情况，OS从物理磁盘上访问读取文件的同时，会顺序对其他相邻块的数据文件进行预读取（ps：顺序读入紧随其后的少数几个页面）。这样，只要下次访问的文件已经被加载至PageCache时，读取操作的速度基本等于访问内存。
2. **对于数据文件的写入**，OS会先写入至Cache内，随后通过异步的方式由pdflush内核线程将Cache内的数据刷盘至物理磁盘上。
    对于文件的顺序读写操作来说，读和写的区域都在OS的PageCache内，此时读写性能接近于内存。**RocketMQ的大致做法是**，将数据文件映射到OS的虚拟内存中（通过JDK NIO的MappedByteBuffer），写消息的时候首先写入PageCache，并通过异步刷盘的方式将消息批量的做持久化（同时也支持同步刷盘）；订阅消费消息时（对CommitLog操作是随机读取），由于PageCache的局部性热点原理且整体情况下还是从旧到新的有序读，因此大部分情况下消息还是可以直接从Page Cache中读取，不会产生太多的缺页（Page Fault）中断而从磁盘读取。

![img](https://pcc.huitogo.club/z0/4325076-9d8c5ba0c12538a6.jpg)

PageCache机制也不是完全无缺点的，当遇到OS进行脏页回写，内存回收，内存swap等情况时，就会引起较大的消息读写延迟。
 对于这些情况，RocketMQ采用了多种优化技术，比如内存预分配，文件预热，mlock系统调用等，来保证在最大可能地发挥PageCache机制优点的同时，尽可能地减少其缺点带来的消息读写延迟。



#### 3. 存储优化技术

##### 3.1 预先分配MappedFile

在消息写入过程中（调用CommitLog的putMessage()方法），CommitLog会先从MappedFileQueue队列中获取一个 MappedFile，如果没有就新建一个。

 这里，MappedFile的创建过程是将构建好的一个AllocateRequest请求（具体做法是，将下一个文件的路径、下下个文件的路径、文件大小为参数封装为AllocateRequest对象）添加至队列中，后台运行的AllocateMappedFileService服务线程（在Broker启动时，该线程就会创建并运行），会不停地run，只要请求队列里存在请求，就会去执行MappedFile映射文件的创建和预分配工作，分配的时候有两种策略，一种是使用Mmap的方式来构建MappedFile实例，另外一种是从TransientStorePool堆外内存池中获取相应的DirectByteBuffer来构建MappedFile（ps：具体采用哪种策略，也与刷盘的方式有关）。并且，在创建分配完下个MappedFile后，还会将下下个MappedFile预先创建并保存至请求队列中等待下次获取时直接返回。**RocketMQ中预分配MappedFile的设计非常巧妙，下次获取时候直接返回就可以不用等待MappedFile创建分配所产生的时间延迟。**



预分配MappedFile的主要过程：

![img](https://pcc.huitogo.club/z0/t1235125123.webp)

##### 3.2 文件预热&&mlock系统调用

1. **mlock系统调用**：其可以将进程使用的部分或者全部的地址空间锁定在物理内存中，防止其被交换到swap空间。对于RocketMQ这种的高吞吐量的分布式消息队列来说，追求的是消息读写低延迟，那么肯定希望尽可能地多使用物理内存，提高数据读写访问的操作效率。
2.  **文件预热**：预热的目的主要有两点；第一点，由于仅分配内存并进行mlock系统调用后并不会为程序完全锁定这些内存，因为其中的分页可能是写时复制的。因此，就有必要对每个内存页面中写入一个假的值。其中，RocketMQ是在创建并分配MappedFile的过程中，预先写入一些随机值至Mmap映射出的内存空间里。第二，调用Mmap进行内存映射后，OS只是建立虚拟内存地址至物理地址的映射表，而实际并没有加载任何文件至内存中。程序要访问数据时OS会检查该部分的分页是否已经在内存中，如果不在，则发出一次缺页中断。这里，可以想象下1G的CommitLog需要发生多少次缺页中断，才能使得对应的数据才能完全加载至物理内存中（ps：X86的Linux中一个标准页面大小是4KB）？**RocketMQ的做法是**，在做Mmap内存映射的同时进行madvise系统调用，目的是使OS做一次内存映射后对应的文件数据尽可能多的预加载至内存中，从而达到内存预热的效果。



#### 4. 刷盘机制

在 Topic 中的 队列是以什么样的形式存在的？

队列中的消息又是如何进行存储持久化的呢？

答案就是通过 **同步刷盘**和 **异步刷盘**进行**磁盘存储**的（当然这里是指**消息日志**）



同步刷盘和异步刷盘流程如下

![img](http://pcc.huitogo.club/9c02c87e013065e6ab1ffa6d0988e26b)

##### 3.1 同步刷盘

同步刷盘中需要等待一个刷盘成功的 ACK ，同步刷盘对 MQ 消息可靠性来说是一种不错的保障，但是 性能上会有较大影响 ，一般地适用于金融等特定业务场景。



但在RocketMq中也对同步刷盘做了点优化，比如

1. 刷盘时使用 FileChannel + DirectBuffer 池，使用堆外内存，加快内存拷贝
2. 使用数据和索引分离，当消息需要写入时，使用 commitlog 文件顺序写，当需要定位某个消息时，查询index 文件来定位，从而减少文件IO随机读写的性能损耗。



##### 3.2 异步刷盘

异步刷盘往往是开启一个线程去异步地执行刷盘操作。消息刷盘采用后台异步线程提交的方式进行， 降低了读写延迟 ，提高了 MQ 的性能和吞吐量，一般适用于如发验证码等对于消息保证要求不太高的业务场景。

异步刷盘能够充分利用OS的PageCache的优势，只要消息写入PageCache即可将成功的ACK返回给Producer端。

一般地，异步刷盘只有在 Broker 意外宕机的时候会丢失部分数据，你可以设置 Broker 的参数 FlushDiskType 来调整你的刷盘策略(ASYNC_FLUSH 或者 SYNC_FLUSH)。