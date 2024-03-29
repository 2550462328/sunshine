Netty 之所以能在高并发的分布式网络环境下实现对数据的快速处理和分发，得益于 其优秀的架构设计。 Netty 架构设计的主要特性有多路复用模型、数据零拷贝、内存重用机制、无锁化设计和高性能的序列化框架。



#### 1. I/O 多路复用模型

在高并发的 I/O编程模型中，一个服务端往往要同时处理成千上万个客户端请求 。 传统的I/O模型会为每个 Socket链路都建立一个线程专门处理该链路上数据的接收和发 送。该方案的特点是，当客户端较少时，服务端的数据处理速度很快，但是当客户端的 数量较多时，服务端会出现没有足够的线程资源为每个 Socket 链路分配一个线程的情况， 服务端会出现大量的线程资源争抢和调度，性能急速下降 。 因此该方案不适合高并发 的场景 。

当有大量客户端并发请求时，可以采用 I/O多路复用模型处理该问题 。I/O多路复用 模型将 I/O的处理分为 Selector 和 Channelo。 Selector 负责多个 Channel 的监控和管理， Channel 负责真正的I/O读写操作 。 当 Selector 监控到 Channel上有数据发生变化时会通 知对应的 Channe l ， Channel 接收到对应的通知后处理对应的数据即可 。 这样一个 Selector 线程可以同时处理多个客户端请求。与传统的多线程I/O模型相比， I/O 多路复用模型的 最大优势是系统开销小，系统不需要为每个客户端连接都建立一个新的线程，也不需要 维护这些线程的运行，减少了系统的维护工作量，节省了系统资源 。

Netty 通过在 NioEventLoop (事件轮询机制)内封装 Selector 来实现I/O 的多路复用， 在一个 NioEventLoop 上服务端可以同时并发处理成千上万个客户端请求，同时由于 Netty 的I/O读写操作都是基于 ByteBuffer 异步非阻塞的，因此大大提高了I/O线程的运 行效率，避免由于频繁I/O阻塞导致的线程挂起和系统资源的浪费。



#### 2. 数据零拷贝

Linux 为了实现操作系统程序和应用程序的资源隔离，将系统分为内核态和用户态 。 操作系统的程序运行在内核态，具体包括进程管理、存储管理 、 文件系统 、 网络通信和 模块机制；应用程序运行在用户态并且不能访问内核态的地址空间，如果应用程序需要访问系统内核态的资源，则需要通过系统调用接口或本地函数库( Libc )来完成。



Linux 的内核结构如图所示

![img](http://pcc.huitogo.club/514565de08f3123e54923985c03c2a41)



具体流程为用户程序调用系统应用接口或本地函数库，系统应用接口或本地函数库调用系统进程获取资源，并将其分配到对应的用户程序 ， 用户程序资源在使用完成后通过系统调用接口或本地函数库释放资源 。

传统的 Socket 服务基于 JVM 堆内存进行 Socke t 读写，也就是说，申请内存资源时， 需要通过 JVM 向操作系统申请堆内存，然后 JVM 将堆内存复制一份到直接内存中，基 于直接内存进行 Socket 读写 。 这样就存在频繁的 JVM 内存数据和 Socket 线程内存数据 来回复制的问题 ， 影响系统性能 。

Netty 的数据接收和发送均采用堆外直接内存进行 Socket 读写，堆外直接内存可以 直接操作系统内存，不需要来回地进行字节缓冲区的二次复制，大大提高了系统的性能 。

同时， Netty 提供了组合 Buffer 对象，基于该对象可以聚合多个 ByteBuffer 对象，使 得用户操作多个 Buffer 与操作一个 Buffer 一样，方便，避免了传统的通过内存复制的方式 将多个小 Buffer 合并为一个大 Buffer 带来的使用不便和性能损耗 。

Netty 的文件传输采用 transferTo 方法 。 transferTo 方法可以直接将文件缓冲区的数据 基于内存映射技术发送到曰标 Channel ，避免了通过循环写方式导致的内存复制问题 。



### 3. 堆外内存

TCP 接收和发送缓冲区采用直接内存代替堆内存，避免了内存复制，提升了 I/O 读取和写入性能。



#### 4. 内存池设计

在 Netty 中，IO 读写必定是非常频繁的操作，而考虑到更高效的网络传输性能，Direct ByteBuffer 必然是最合适的选择。但是 Direct ByteBuffer 的申请和释放是高成本的操作，那么进行**池化**管理，多次重用是比较有效的方式。但是，不同于一般于我们常见的对象池、连接池等**池化**的案例，ByteBuffer 是有**大小**一说。又但是，申请多大的 Direct ByteBuffer 进行池化又会是一个大问题，太大会浪费内存，太小又会出现频繁的扩容和内存复制！！！所以呢，就需要有一个合适的内存管理算法，解决**高效分配内存**的同时又解决**内存碎片化**的问题。



**官方的说法**

> Netty 4.x 增加了 Pooled Buffer，实现了高性能的 buffer 池，分配策略则是结合了 buddy allocation 和 slab allocation 的 **jemalloc** 变种，代码在`io.netty.buffer.PoolArena` 中。
>
> 官方说提供了以下优势：
>
> - 频繁分配、释放 buffer 时减少了 GC 压力。
> - 在初始化新 buffer 时减少内存带宽消耗( 初始化时不可避免的要给buffer数组赋初始值 )。
> - 及时的释放 direct buffer 。



#### 5. 无锁化设计

对于一般程序来说，多线程会提高系统的并发度，但是线程数并不是越多越好，过 多的线程会引起 CPU 的频繁切换而增加系统的负载。 Netty 内部采用串行无锁化设计的 思想对I/O进行操作，避免多线程竞争 CPU 和资源锁定导致的性能下降。在具体的使用 中，可以调整 NIO 线程池的线程参数 ，同时启动多个串行化的线程并行运行，这种局部 无锁化的串行多线程设计比一个队列结合多个工作线程模型的性能更佳。

Netty 的 NioEventLoop 的设计思路是 Channel 读取消息后，直接调用 ChannelPipeline 的 fireChannelRead(Object msg)进行消息的处理，如果在运行过程中用户不主动切换线程， Netly 的 NioEventLoop 则一直在该线程上进行消息的处理，这种线程绑定 CPU 持续执行 的方式可以有效减少系统资源的竞争和切换，对于持续高并发的操作来说性能有很大的 提升 。



#### 5. 高性能的序列化框架

Netty 默认基于 Google ProtoBuf 实现数据的序列化，通过扩展 Netty 的编解码接口， 用户可以实现自定义的序列化框架，例如 Thrift 的压缩 二进制编解码框架 。



在使用 ProtoBuf 序列化的时候需要注意以下几点。

- SO_RCVBUF 和 SO_SNDBUF 的设置：SO_RCVBUF 为接收数据的 Buffer 大小， SO_SNDBUF 为发送数据的 Buffer 大小，通常建议值为 128KB 或者 256KB 。
- SO_TCPNODELAY 的设置：将 SO_TCPNODELAY 设置为 true ，表示开启自动 粘包操作，该操作采用 Nagle 算法将缓冲区内字节较少的包前后相连组成一个大包一起 发送，防止大量小包频繁发送造成网络的阻塞，从而提高网络的吞吐量 。该算法对单条 数据延迟的要求不高，但在传输数据最大的情况下能显著提高性能 。
- 软中断：在开启软中断后， Netty 会根据数据包的源地址 、 源端口、目的地址 、 目的端口计算一个 Hash 值，根据该Hash 值选择对应的 CPU 执行软中断 。 即 Netty 将每 个连接都与 CPU 绑定 ， 并通过 Hash 值在多个 CPU 上均衡软中断，以提高 CPU 的性能 。



#### 6. 内存泄露检测

通过引用计数器及时地释放不再被引用的对象，细粒度的内存管理降低了 GC 的频率，减少频繁 GC 带来的时延增大和 CPU 损耗。