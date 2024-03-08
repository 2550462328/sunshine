## 1. 连接篇

### 1.1 Netty Native

Netty Native用C++编写JNI调用的Socket Transport，是由Twitter将Tomcat Native的移植过来，现在还时不时和汤姆家同步一下代码。

经测试，的确比JDK NIO更省CPU。

也许有人问，JDK的NIO也用EPOLL啊，大家有什么不同？ Norman Maurer这么说的：

- Netty的 epoll transport使用 edge-triggered ，而 Java NIO 用 level-triggered
- C代码，更少GC，更少synchronized
- 暴露了更多的Socket配置参数



用法倒是简单，只要几个类名替换一下，详见Netty的官方文档。

但要注意，它跟OS相关且基于GLIBC2.10编译，而CentOS 5.8就只有GLIBC2.5（别问为什么，厂大怪事多，我厂就是还有些CentOS5.8的机器），所以最好还是不要狠狠的直接全文搜索替换，而是用System.getProperty(“os.name”) 和 System.getProperty(“os.version”) 取出操作系统名称与版本，做成一个开关。

另外，Netty很多版本都有修复Netty Native相关的bug，看得人心里发毛，好在最近的版本终于不再说了，所以要用就用Netty的新版。

最后，Netty Native还包含了Google的boringssl(A fork of OpenSSL)，JDK的原生SSL实现比OpenSSL慢很多，而大家把SSL Provider配置成OpenSSL时，又要担心操作系统有没装OpenSSL，或者版本会不会太旧。现在好了。



### 1.2 异步连接，异步传输，告别Commons Pool

异步化最牛头不对马嘴的事情就是，给它配一个类似Commons Pool这样，有借有还的连接池。

在异步化的场景里，用channel.writeAndFlush()原子地发送数据，发完不用同步等response的话，其实不需要独占一条Channel，不需要把它借出来，再还回池里。一来连接池出入之间有并发锁，二来并发请求一多就要狂建连接，到了连接池上限时还要傻傻的等待别人释放连接，而这可能毫无必要。

此时，建议直接建一个连接数组，随机到哪个连接就直接用它发送数据。如果那个连接还没建立或者已经失效，那就建立连接。

顺便说一句，异步的世界里，连建立连接的过程也是异步的，主线程不要等在建连接上，而是把发送的动作封成一个ChannelCallback，等连接建立了，再回调它发送数据，避免因为连接建立的缓慢或网络根本不通，把线程都堵塞了。

Netty4.0.28开始也有ChannelPool了，供需要独占Channel的场景如HTTP1.1，比之Commons Pool的特色之一也是这个异步的建连接过程。



### 1.3 最佳连接数：一条连接打天下？还有传说中的海量连接？

NIO这么神奇，有一种做法是只建一条连接，如SpyMemcached。还有一种是既然你能支持海量连接，几千几万的，那我就无节制的可劲的建了。

测试表明，一条连接有瓶颈，毕竟只用到了一个CPU核。 海量连接，CPU和内存在燃烧。。。。

那最佳连接数是传说中的CPU核数么？依然不是。

一切还是看你的场景，连接数在满足传输吞吐量的情况下，越少越好。



举个例子，在我的Proxy测试场景里：

- 2条连接时，只能有40k QPS。
- 48条连接，升到62k QPS，CPU烧了28%
- 4条连接，QPS反而上升到68k ，而且CPU降到20%。



### 1.4 Channel参数设定

- 1.4.1 TCP/Socket参数

TCP/Socket的大路设置，无非 SO_REUSEADDR， TCP_NODELAY， SO_KEEPALIVE 。另外还有SO_LINGER ， SO_TIMEOUT, SO_BACKLOG, SO_SNDBUF, SO_RCVBUF。

而用了Native后又加了TCP_CORK和KeepAlive包发送的时间间隔(默认2小时)，详见EpoolSocketChannelConfig的JavaDoc。

- 1.4.2 Netty参数

CONNECT_TIMEOUT_MILLIS，Netty自己起一个定时任务来监控建立连接是否超时，默认30秒太长谁也受不了，一般会弄短它。

WRITE_BUFFER_HIGH_WATER_MARK 与 WRITE_BUFFER_LOW_WATER_MARK是两个流控的参数，默认值分别为32*2K与32K。如果在write buffer里排队准备输出的字节超过上限，Channel就不是writable的，NIO的事件轮询里就会把它摘掉，直到它低于32k才重新变回writable。建议没有足够的测试不要动它。



## 2. 线程篇

基本知识：《Netty in Action》中文版—第七章 EventLoop和线程模型



### 2.1 WorkerGroup 与 Boss Group

大家都知道，Boss Group用于服务端处理建立连接的请求，WorkGroup用于处理I/O。

EventLoopGroup的默认大小都是是2倍的CPU核数，但这并不是一个恒定的最佳数量，为了避免线程上下文切换，只要能满足要求，这个值其实越少越好。

如果都是长连接，Boss Group平时很闲，好在它也只有忙起来才会多起线程，平时就只占1条。



### 2.2 上下游线程的绑定

在服务化的应用里，一般处理上游请求的同时，也会向多个下游的服务集群发送请求，但调优指南里都说，尽量，全部重用同一个EventLoopGroup。否则，处理上游请求的线程，就要把后续任务以Runnable的方式，提交到下游Channel的处理线程。

但，一个EventLoop线程可以处理多个Channel的信息，而一个Channel只能注册一个EventLoop线程。所以没办法保证处理上游的Channel，与下游多个连接的Channel，刚好是属于一个EventLoop？

因此，追求极致的Proxy型应用，可能会放弃前面的固定连接池的做法，而是为每个处理上游请求的线程，对应每一台下游服务器创建一条Channel，而且设定它的工作线程就是本上游线程，然后存到threadLocal里。这样的做法连接数可能会增多，但减少了切换，要自行测试权衡。



### 2.3 业务线程池

Netty线程的数量一般固定且较少，所以很怕线程被堵塞，比如同步的数据库查询，比如下游的服务调用（又来罗嗦，future.get()式的异步在执行future.get()时还是堵住当前线程的啊）。

所以，此时就要把处理放到一个业务线程池里操作，即使要付出线程上下文切换的代价，甚至还有些ThreadLocal需要复制。



### 2.4 定时任务

像发送超时控制之类的一次性任务，不要使用JDK的ScheduledExecutorService，而是如下：

```
 ctx.executor().schedule(new MyTimeoutTask(p), 30, TimeUnit.SECONDS)
```

首先，JDK的ScheduledExecutorService是一个大池子，多线程争抢并发锁。而上面的写法，TimeoutTask只属于当前的EventLoop，没有任何锁。

其次，如果发送成功，需要从长长Queue里找回任务来取消掉它。现在每个EventLoop一条Queue，明显长度只有原来的N分之一。



### 2.5 快速复习一下Netty的高性能线程池

Netty的线程池理念有点像ForkJoinPool，都不是一个线程大池子并发等待一条任务队列，而是每条线程自己一个任务队列，怎么做的？建了N个只有一条线程的线程池。

而且Netty的线程，并不只是简单的阻塞地拉取任务，而是非常辛苦命的在每个循环做三件事情：

1. 先SelectKeys()处理NIO的事件
2. 然后获取2.3里提到的本线程的定时任务，放到本线程的任务队列里
3. 再然后混合2.2里提到的其他线程提交给本线程的任务，一起执行



每个循环里处理NIO事件与其他任务的时间消耗比例，还能通过ioRatio变量来控制，默认是各占50%。

可见，Netty的线程根本没有阻塞等待任务的清闲日子，所以也不使用有锁的BlockingQueue如ArrayBlockingQueue来做任务队列了，而是使用后面提到的无锁的MpscLinkedQueue(Mpsc 是Multiple Producer, Single Consumer的缩写)。



## 3. 内存篇

### 3.1 堆外内存池

堆外内存是Netty被说得最多的部分，网卡内核态与应用用户态之间零复制啊，无GC啊，不受堆内存大小限制啊，不重复。

内存池的算法也是Netty骄傲的地方（注意，在4.0的刚开始版本这也是经常改的）

Norman Maurer说，只有在输出时要需要编码对象直接操作bytes[]时，才有可能用回Heap Buffer。

- 3.1.1 ByteBuf释放

各种异常处理，一不留神，我的踩坑之作:Netty之有效规避内存泄漏

建议写足够的单元测试，在测试里将内存泄漏检查级别开到最高，然后每个用例执行完就System.gc()一次，同时加入一个测试用的log appender监控Netty的logger有没有输出memory leak信息。

如果信心已足，在生产环境里，就可以加上”-Dio.netty.leakDetectionLevel=disabled”把检测关掉，提高那么点点理论上存在的性能。

- 3.1.2 Recycler

Netty的另一个得意设计是对象可以在线程内无锁的被回收重用。但有些场景里，某线程只创建对象，要到另一个线程里释放，一不小心，你就会发现应用缓缓变慢，heap dump时看到好多RecyleHandler对象。所以这块的设计其实在4.0.x的各个版本里变动了无数遍，貌似4.0.40版才终于在我的测试里不再泄漏了。

但有时觉得这么辛苦的重用一个对象（而不是ByteBuffer内存本身），不如干脆禁止掉这个功能，所以4.0.0.33里，我会在启动参数里加入 -Dio.netty.recycler.maxCapacity.default=0。无语的是，也几乎从这个版本开始，才能通过设为0禁止它。



### 3.2 避免复制：CompositeByteBuff, slice(), duplicate()

尽量，尽量不要进行ByteBuf内容复制。

场景1: 为了失败时重试，我要保留内容稍后使用，不想Netty在发送完毕后把内容释放了，最笨的方法是用copy()复制一个新的ByteBuf。

Bytebuf newBuf = oldBuf.duplicate().retain();
而上句只是复制出独立的读写Index, 而底下的ByteBuffer是共享的，同时将ByteBuffer的计数器＋1.

场景2: 在Proxy型的应用里，输入输出的内容不变，只替换一些头信息。

聪明的做法是，用slice().retain()语句从旧的ByteBuf中切割出Header外的Body部分，同样是共享底层ByteBuffer。然后生成一个新的Header，然后用CompositeByteBuff，将新的Header 与 旧的Body拼接起来。



### 3.3 避免扩容: ByteBuf的大小预估与AdaptiveRecvByteBufAllocator

ByteBuf如果一开始申请的不足，到极限时会智能的扩容，但也和Java一样，需要重新申请两倍的内存，然后把旧的内容复制过去，一听就是个很消耗的动作，因此，反正是堆外内存池，一开始还是给多一点吧。

另一个有趣的思路是Netty的自适应算法。Netty收到一个请求时，什么都不知道啊，那会申请多大的内存来接收它呢？在Bootstrap里可以配置，默认是 AdaptiveRecvByteBufAllocator，根据每一次收到的请求动态变化。

但如果一个应用有几个不同接口，请求的大小变来变去，会不会玩死它呢？好像会的。不过服务化体系里的特征都是请求小，返回大，请求包的大小变化不会太剧烈。



### 3.4 烦人的rangeChecking

Norman Maurer说，如果你要搜索某个Byte是否存在，请用 byteBuf.forEachByte(ByteProcessor processor), 比循环的遍历地调用byteBuf.readByte()要快得多。原因无它，ByteBuf有Java其他集合同样的rangeChecking。

每次readByte()都不是读一个字节这么简单，首先要判断refCnt()>0，然后再做范围检查防止越界。getByte(i＝int)更加一层又一层的检查函数，JVM没有帮你内联或者Profiler工具阻止了你的内联的话，够呛。



### 3.5 readInt()，不要readBytes(bytes[],0,4)

比如Thrift，它会做一层封装，先用byteBuf.readBytes(bytes,0,4)读取4个字节的byte[]，再自己转成int。

但实测证明，我将所有的读写short, int, long, boolean, byte的函数，改造为直接使用Netty的原生函数时，性能从7万QPS提升到7.4万QPS，而消耗CPU不变。



### 3.6 对String说不的 ASCIIString

Netty收到的bytes[]，大部分时候最终都要变回String。但String的内部是char[]啊，出入都要经过CharsetEncoder进行转换成byte[]，既浪费CPU，又浪费内存。

ByteBufUtils类提供了写入UTF-8和ASCII的优化，不需要从String编码出一个新的bytes[]再开始写入ByteBuf，而是遍历一个个char，当场编码当场写入。可惜此优化对于thrift这种需要先得到byte[]长度的无效。

而Netty 4.1开始，提供了实现 CharSequence接口的ASCIIString。原理就是，String要存char[]，是因为UTF-8这样的不定长Encoder，会把char转成1～3个byte。但如果我的Header的名称与某些值，肯定是ASCII字符时（比如服务名，服务版本），那一个char只对应一个byte啊，那你直接在构造函数里把byte[]交给我内部存起来就行了啊，不需要转换成char[]啊，不费CPU又不废内存了啊。



## 4. 工具类篇

Netty 为了高效编程，或写或借，搞了一些高效的工具类，在自己的应用里同样可以借用一下哦。

Netty自己有一篇Using as a generic library 介绍了其中的一些。本文主要介绍与性能相关的。



### 4.1 FastThreadLocal

Netty威武，居然太岁头上动土，搞出个比JDK的ThreadLocal还快的ThreadLocal。详见Netty精粹之设计更快的ThreadLocal

JDK的ThreadLocal，实现原理是Thread对象里有个HashMap式的数组，每个ThreadLocal的id是个Hashcode，算法是currentValue+0x61c88647，hashCode取模数组大小得到threadLocal存的位置，如果桶里已有其他元素，key.nextHashCode()找下一个桶，小学学过的HashMap实现之一，开放地址式就不重复了。

而FastThreadLocal的id则是一个自增的int，FastThreadLocalThread里放一个数组，直接按下标获取，没有hash，没有比较，没有冲突。不过需要在Netty地界里用，业务线程池就要自己定义ThreadFactory，创建FastThreadLocalThread 而不是Thread。



### 4.2 移植JDK8的宝贝到JDK7

JDK8重写了ConcurrentHashMap，原来的Load Factor，Current Level都没有作用了。

ThreadLocalRandom就是把原来有全局锁的Random，通过ThreadLocal化取消了锁。

LongAdder则是把AtomicLong打散成几个，平时＋＋的时候找其中一个执行，减少CMS冲突的概率，等get()的时候才把几个counter累计起来，适合increment()多，get()少的情况。

Netty把这些类都复制黏贴了一份，封装在 PlatformDependent里，根据JDK版本决定返回JDK原生的还是它复制的。



### 4.3 其他宝贝

- 4.3.1 ThreadLocal的StringBuilder

之前写过StringBuilder在高性能场景下的正确用法 ，才发现Netty和我做了同样的事，通过ThreadLocal的保存，重用StringBuilder对象，节约内存和分配内存的时间。当然，如果字符串只是很短就未必有必要。

- 4.3.2 IntHashMap

原始类型的map，比如key是int而不是Integer的Map，在某些次元里挺流行的，Trove，Koloboke, FastUtil等等，好处一是int比Integer省地方，int是4bytes，Integer是12+4 bytes，另外数据结构与解决冲突的方式也完全不同，更快更省，详见高性能场景下，关于HashMap的补课

比起FastUtils.jar 穷举各种原始类型-原始类型/对象类型的组合，动不动10M大小。Netty只有IntHashMap一个类, 4.1又增加了LongHashMap等，够用了。

- 4.3.3 JCTools的Queue

针对Multiple Producer－Single Consumer，Single Producer－Multiple Consumer等不同场景专门设计，做到最少的锁。
不过它并不提供阻塞等待的函数，所以不能拿来替换ArrayBlockingQueue。

- 4.3.4 RecyclableArrayList

如果你需要经常创建很长的ArrayList，不想浪费了，可以考虑用它来节约GC，不过到底哪边的代价大，一定要真正测试后决定。详见Netty精粹之轻量级内存池技术实现原理与应用。



## 5. 其他零碎篇

主要来自Norman Maurer的文章：

1. ctx.writeAndFlush() 与 channel.writeAndFlush()的区别在于，channel要经过整条Pipeline，而ctx直接找下一个outboundHandler。
2. channel.writeAndFlush(buf, channel.voidPromise() )
   writeAndFlush不管你用不用默认构造返回一个Promise(Future)，有点浪费内存。没有用的话，用一个公共的 void Promise ，减少大家花费。不过低版本的Netty不能用。
3. 空闲连接管理，因为刚才说的ctx.writeAndFlush()不经Pipeline，所以只监控读空闲就够了。否则每次请求都要READ/WRITE/ALL IDEL三个值算一遍，白白消耗。
4. writeAndFlush()不要太多，毕竟调用了系统调用。
5. Handler能共用就标上Shareable Annotation然后共用，不要每个Channel建一个。