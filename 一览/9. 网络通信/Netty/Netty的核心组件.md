核心组件的交互流程：

![img](http://pcc.huitogo.club/5ce22b0d684a5d69f0394c841be076c9)





核心组件的高层类图：

![高层类图](http://static.iocoder.cn/images/Netty/2018_03_05/01.png)



#### 1. Bytebuf（字节容器）

网络通信最终都是通过字节流进行传输的。 ByteBuf 就是 Netty 提供的一个字节容器，其内部是一个字节数组。

我们可以将 ByteBuf 看作是 Netty 对 Java NIO 提供了 ByteBuffer 字节容器的封装和抽象。



至于**为什么使用ByteBuf？**

如下是 《Netty 实战》对它的**优点**总结：

> - A01. 它可以被用户自定义的**缓冲区类型**扩展
> - A02. 通过内置的符合缓冲区类型实现了透明的**零拷贝**
> - A03. 容量可以**按需增长**
> - A04. 在读和写这两种模式之间切换不需要调用 `#flip()` 方法
> - A05. 读和写使用了**不同的索引**
> - A06. 支持方法的**链式**调用
> - A07. 支持引用计数
> - A08. 支持**池化**



特别是A04这点，我们知道传统的NIO利用ByteBuffer读写前都要进行复位（flip）,因为它只有一个指针。

![img](http://pcc.huitogo.club/93fffa072bb50059302bada39b33db5a)



而**ByteBuf则不需要复位，因为它有读写两个指针**。

需要注意的是**ByteBuf资源需要手动去释放**。



***那么为什么ByteBuf需要手动释放？***

Netty 的 ByteBuf 带给我们的最大不同，就是他不再基于传统 JVM 的 GC 模式，相反，它采用了类似于 C++ 中的 malloc/free 的机制，需要开发人员来手动管理回收与释放。



#### 2. Bootstrap 和 ServerBootstrap

这 2 个类都继承了AbstractBootstrap，因此它们有很多相同的方法和职责。它们都是**启动器**，能够帮助 Netty 使用者更加方便地组装和配置 Netty ，也可以更方便地启动 Netty 应用程序。相比使用者自己从头去将 Netty 的各部分组装起来要方便得多，降低了使用者的学习和使用成本。它们是我们使用 Netty 的入口和最重要的 API ，可以通过它来连接到一个主机和端口上，也可以通过它来绑定到一个本地的端口上。总的来说，**它们两者之间相同之处要大于不同**。



它们和其它组件之间的关系是它们将 Netty 的其它组件进行组装和配置，所以它们会组合和直接或间接依赖其它的类。

Bootstrap 用于启动一个 Netty TCP 客户端，或者 UDP 的一端。

- 通常使用 `#connet(...)` 方法连接到远程的主机和端口，作为一个 Netty TCP 客户端。
- 也可以通过 `#bind(...)` 方法绑定本地的一个端口，作为 UDP 的一端。
- 仅仅需要使用**一个** EventLoopGroup 。

ServerBootstrap 往往是用于启动一个 Netty 服务端。

- 通常使用 `#bind(...)` 方法绑定本地的端口上，然后等待客户端的连接。

- 使用**两个** EventLoopGroup 对象( 当然这个对象可以引用同一个对象 )：第一个用于处理它本地 Socket **连接**的 IO 事件处理，而第二个责负责处理远程客户端的 IO 事件处理。

  

比如ServerBootstrap 配置

```
1.ServerBootstrap b =new ServerBootstrap(); // 创建辅助工具类，用于服务器端通道的一系列配置 

2. b.group(pGroup, cGroup) // 绑定两个线程组 

3.   .channel(NioServerSocketChannel.class) // 指定NIO的模式 

4.   .option(ChannelOption.SO_BACKLOG, 1024) // 指定tcp的缓冲区 

5.   .option(ChannelOption.SO_SNDBUF, 32 * 1024) // 设置发送的缓冲大小 

6.   .option(ChannelOption.SO_RCVBUF, 32 * 1024) // 设置接收的缓冲大小 

7.   .option(ChannelOption.SO_KEEPALIVE, true) // 保持连接 

8.   .childHandler(new ChannelInitializer<SocketChannel>() { 

9.     @Override 

10.     protected void initChannel(SocketChannel sc) throws Exception { 

11.       sc.pipeline().addLast(new ServerHandler()); // 在这里设置具体数据接收方法的处理 

12.     } 

13.   }); 
```



#### 3. Channel（网络操作抽象类）

Channel 是 Netty 网络操作抽象类，它除了包括基本的 I/O 操作，如 bind、connect、read、write 之外，还包括了 Netty 框架相关的一些功能，如获取该 Channel 的 EventLoop 。

在传统的网络编程中，作为核心类的 Socket ，它对程序员来说并不是那么友好，直接使用其成本还是稍微高了点。而 Netty 的 Channel 则提供的一系列的 API ，它大大降低了直接与 Socket 进行操作的复杂性。而相对于原生 NIO 的 Channel，Netty 的 Channel 具有如下优势( 摘自《Netty权威指南( 第二版 )》) ：

- 在 Channel 接口层，采用 Facade 模式进行统一封装，将网络 I/O 操作、网络 I/O 相关联的其他操作封装起来，统一对外提供。
- Channel 接口的定义尽量大而全，为 SocketChannel 和 ServerSocketChannel 提供统一的视图，由不同子类实现不同的功能，公共功能在抽象父类中实现，最大程度地实现功能和接口的重用。
- 具体实现采用聚合而非包含的方式，将相关的功能类聚合在 Channel 中，由 Channel 统一负责和调度，功能实现更加灵活。



我们可以指定Channel类型

```
1. b.group(group).channel(NioSocketChannel.class).handler(new ChannelInitializer<SocketChannel>() { 

2.  

3.   @Override 

4.   protected void initChannel(SocketChannel sc) throws Exception { 

5.     sc.pipeline().addLast(new ClientHandler()); 

6.   } 

7. }); 
```



一般都会用NioSocketChannel去做异步操作

然后当我们连接Server端的时候会返回异步的Channel对象，而我们可以通过它读写数据

```
1. ChannelFuture cf1 = b.connect("127.0.0.1", 8765).sync(); 

2. cf1.channel().writeAndFlush(Unpooled.copiedBuffer("hello netty".getBytes())); 

4. cf1.channel().closeFuture().sync(); 
```



#### 4. EventLoop和EventLoopGroup



EventLoop 定义了 Netty 的核心抽象，用于处理连接的生命周期中所发生的事件。

EventLoop 的主要作用**实际就是责监听网络事件并调用事件处理器进行相关 I/O 操作（读写）的处理。**你可以把它想象成一个不断处理任务的线程。

EventLoopGroup 是一个 EventLoop 的分组，它可以获取到一个或者多个 EventLoop 对象，因此它提供了迭代出 EventLoop 对象的方法。



客户端会配置一个EventLooGroup去做读写操作，而服务端会配置两个，因为它需要一个EvenLoopGroup去处理客户端连接。

```
1. EventLoopGroup pGroup = new NioEventLoopGroup(); // 用来处理服务器端接受客户端连接的 

2. EventLoopGroup cGroup = new NioEventLoopGroup(); // 用来进行网络读写的 
```



**Channel 和 EventLoop 的关系**

Channel 为 Netty 网络操作(读写等操作)抽象类，EventLoop 负责处理注册到其上的Channel 的 I/O 操作，两者配合进行 I/O 操作。



下图是 Channel、EventLoop、Thread、EventLoopGroup 之间的关系( 摘自《Netty In Action》) ：

![Channel、EventLoop、Thread、EventLoopGroup](http://static.iocoder.cn/images/Netty/2018_03_05/02.png)

- 一个 EventLoopGroup 包含一个或多个 EventLoop ，即 EventLoopGroup : EventLoop = `1 : n` 。
- 一个 EventLoop 在它的生命周期内，只能与一个 Thread 绑定，即 EventLoop : Thread = `1 : 1` 。
- 所有有 EventLoop 处理的 I/O 事件都将在它**专有**的 Thread 上被处理，从而保证线程安全，即 Thread : EventLoop = `1 : 1`。
- 一个 Channel 在它的生命周期内只能注册到一个 EventLoop 上，即 Channel : EventLoop = `n : 1` 。
- 一个 EventLoop 可被分配至一个或多个 Channel ，即 EventLoop : Channel = `1 : n` 。



**EventLoop怎么保证线程安全？**

EventLoop 处理的 I/O 事件都将在它专有的 Thread 上被处理，即 Thread 和 EventLoop 属于 1 : 1 的关系，从而保证线程安全。

![img](http://pcc.huitogo.club/a7cb59fc286591a8fb419b883dff1a43)



#### 5. ChannelHandler 

ChannelHandler ，连接通道处理器，我们使用 Netty 中**最常用**的组件。ChannelHandler 主要用来处理各种事件，这里的事件很广泛，比如可以是连接、数据接收、异常、数据转换等。

ChannelHandler 有两个核心子类 ChannelInboundHandler 和 ChannelOutboundHandler，其中 ChannelInboundHandler 用于接收、处理入站( Inbound )的数据和事件，而 ChannelOutboundHandler 则相反，用于接收、处理出站( Outbound )的数据和事件。

- ChannelInboundHandler 的实现类还包括一系列的 **Decoder** 类，对输入字节流进行解码。
- ChannelOutboundHandler 的实现类还包括一系列的 **Encoder** 类，对输入字节流进行编码。

ChannelDuplexHandler 可以**同时**用于接收、处理入站和出站的数据和时间。

ChannelHandler 还有其它的一系列的抽象实现 Adapter ，以及一些用于编解码具体协议的 ChannelHandler 实现类。



#### 6.  ChannelPipeline

ChannelPipeline 为 ChannelHandler 的**链**，提供了一个容器并定义了用于沿着链传播入站和出站事件流的 API 。一个数据或者事件可能会被多个 Handler 处理，在这个过程中，数据或者事件经流 ChannelPipeline ，由 ChannelHandler 处理。在这个处理过程中，一个 ChannelHandler 接收数据后处理完成后交给下一个 ChannelHandler，或者什么都不做直接交给下一个 ChannelHandler。

![ChannelPipeline](http://static.iocoder.cn/images/Netty/2018_03_05/03.png)

- 当一个数据流进入 ChannelPipeline 时，它会从 ChannelPipeline 头部开始，传给第一个 ChannelInboundHandler 。当第一个处理完后再传给下一个，一直传递到管道的尾部。
- 与之相对应的是，当数据被写出时，它会从管道的尾部开始，先经过管道尾部的“最后”一个ChannelOutboundHandler ，当它处理完成后会传递给前一个 ChannelOutboundHandler 。



上图更详细的，可以是如下过程：

```
*                                                 I/O Request
*                                            via {@link Channel} or
*                                        {@link ChannelHandlerContext}
*                                                      |
*  +---------------------------------------------------+---------------+
*  |                           ChannelPipeline         |               |
*  |                                                  \|/              |
*  |    +---------------------+            +-----------+----------+    |
*  |    | Inbound Handler  N  |            | Outbound Handler  1  |    |
*  |    +----------+----------+            +-----------+----------+    |
*  |              /|\                                  |               |
*  |               |                                  \|/              |
*  |    +----------+----------+            +-----------+----------+    |
*  |    | Inbound Handler N-1 |            | Outbound Handler  2  |    |
*  |    +----------+----------+            +-----------+----------+    |
*  |              /|\                                  .               |
*  |               .                                   .               |
*  | ChannelHandlerContext.fireIN_EVT() ChannelHandlerContext.OUT_EVT()|
*  |        [ method call]                       [method call]         |
*  |               .                                   .               |
*  |               .                                  \|/              |
*  |    +----------+----------+            +-----------+----------+    |
*  |    | Inbound Handler  2  |            | Outbound Handler M-1 |    |
*  |    +----------+----------+            +-----------+----------+    |
*  |              /|\                                  |               |
*  |               |                                  \|/              |
*  |    +----------+----------+            +-----------+----------+    |
*  |    | Inbound Handler  1  |            | Outbound Handler  M  |    |
*  |    +----------+----------+            +-----------+----------+    |
*  |              /|\                                  |               |
*  +---------------+-----------------------------------+---------------+
*                  |                                  \|/
*  +---------------+-----------------------------------+---------------+
*  |               |                                   |               |
*  |       [ Socket.read() ]                    [ Socket.write() ]     |
*  |                                                                   |
*  |  Netty Internal I/O Threads (Transport Implementation)            |
*  +-------------------------------------------------------------------+
```



当 ChannelHandler 被添加到 ChannelPipeline 时，它将会被分配一个 **ChannelHandlerContext** ，它代表了 ChannelHandler 和 ChannelPipeline 之间的绑定。其中 ChannelHandler 添加到 ChannelPipeline 中，通过 ChannelInitializer 来实现，过程如下：

1. 一个 ChannelInitializer 的实现对象，被设置到了 BootStrap 或 ServerBootStrap 中。

2. 当 `ChannelInitializer#initChannel()` 方法被调用时，ChannelInitializer 将在 ChannelPipeline 中创建**一组**自定义的 ChannelHandler 对象。

3. ChannelInitializer 将它自己从 ChannelPipeline 中移除，因为**ChannelInitializer 是一个特殊的 ChannelInboundHandlerAdapter 抽象类**。

   

#### 7. ChannelFuture（操作执行结果）

```
1. public interface ChannelFuture extends Future<Void> { 

2.   Channel channel(); 

3.  

4.   ChannelFuture addListener(GenericFutureListener<? extends Future<? super Void>> var1); 

5.   ...... 

6.  

7.   ChannelFuture sync() throws InterruptedException; 

8. } 
```



Netty 是异步非阻塞的，所有的 I/O 操作都为异步的。

因此，我们不能立刻得到操作是否执行成功，但是，你可以通过 ChannelFuture 接口的 addListener() 方法注册一个 ChannelFutureListener，当操作执行成功或者失败时，监听就会自动触发返回结果。

```
1. ChannelFuture f = b.connect(host, port).addListener(future -> { 

2.  if (future.isSuccess()) { 

3.   System.out.println("连接成功!"); 

4.  } else { 

5.   System.err.println("连接失败!"); 

6.  } 

7. }).sync(); 
```



并且，你还可以通过ChannelFuture 的 channel() 方法获取连接相关联的Channel 。

另外，我们还可以通过 ChannelFuture 接口的 sync()方法让异步的操作编程同步的。

```
1. //bind()是异步的，但是，你可以通过 `sync()`方法将其变为同步。 

2. ChannelFuture f = b.bind(port).sync(); 
```