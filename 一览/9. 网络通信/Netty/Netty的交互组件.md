![img](http://pcc.huitogo.club/5ce22b0d684a5d69f0394c841be076c9)



核心组件的交互流程在上面，实际还有一些细节

#### 1. Bytebuf（字节容器）

网络通信最终都是通过字节流进行传输的。 ByteBuf 就是 Netty 提供的一个字节容器，其内部是一个字节数组。

我们可以将 ByteBuf 看作是 Netty 对 Java NIO 提供了 ByteBuffer 字节容器的封装和抽象。



至于**为什么使用ByteBuf？**

可以看作netty的一个优化



传统的NIO利用ByteBuffer读写前都要进行复位（flip）,因为它只有一个指针。

![img](http://pcc.huitogo.club/93fffa072bb50059302bada39b33db5a)



而**ByteBuf则不需要复位，因为它有读写两个指针**。

需要注意的是**ByteBuf资源需要手动去释放**。

因为ByteBuf是一个计数对象，而处理器的职责需要释放所有传递到处理器的计数对象。但是如果在业务代码中执行写操作时使用writeAndflush方法则不用手动释放资源，因为它会自动去释放。



#### 2. Bootstrap 和 ServerBootstrap

可以看出是客户端和服务端的引导类，我更倾向于叫它配置类。



比如服务端配置

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

Channel 接口是 Netty 对网络操作抽象类。通过 Channel 我们可以进行 I/O 操作。

比如我们熟悉的NIO中的FileChannel，就是操作对文件的读写。



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

3.  

4. cf1.channel().closeFuture().sync(); 
```



#### 4. EventLoop（事件循环）

EventLoop 定义了 Netty 的核心抽象，用于处理连接的生命周期中所发生的事件。

EventLoop 的主要作用**实际就是责监听网络事件并调用事件处理器进行相关 I/O 操作（读写）的处理。**

你可以把它想象成一个不断处理任务的线程。而EventLoopGroup相当于线程池。



客户端会配置一个EventLooGroup去做读写操作，而服务端会配置两个，因为它需要一个EvenLoopGroup去处理客户端连接。

```
1. EventLoopGroup pGroup = new NioEventLoopGroup(); // 用来处理服务器端接受客户端连接的 

2. EventLoopGroup cGroup = new NioEventLoopGroup(); // 用来进行网络读写的 
```



**Channel 和 EventLoop 的关系**

Channel 为 Netty 网络操作(读写等操作)抽象类，EventLoop 负责处理注册到其上的Channel 的 I/O 操作，两者配合进行 I/O 操作。



**EventLoop怎么保证线程安全？**

EventLoop 处理的 I/O 事件都将在它专有的 Thread 上被处理，即 Thread 和 EventLoop 属于 1 : 1 的关系，从而保证线程安全。

![img](http://pcc.huitogo.club/a7cb59fc286591a8fb419b883dff1a43)



#### 5. ChannelHandler 和 ChannelPipeline

就是对消息的处理，当然肯定不是只有一个处理，netty这里使用的是链式的处理方法，可以联想到拦截器模型。



比如对消息处理之前还有处理一下粘包问题和编码问题

```
1. b.group(pGroup, cGroup) //绑定两个线程组 

2. .channel(NioServerSocketChannel.class) //指定NIO的模式 

3. .childHandler(new ChannelInitializer<SocketChannel>() { 

4.   @Override 

5.   protected void initChannel(SocketChannel sc) throws Exception { 

6.     //设置分隔符 

7.     ByteBuf buf = Unpooled.copiedBuffer("$_".getBytes()); 

8.     sc.pipeline().addLast(new FixedLengthFrameDecoder(5)); //固定读取五个长度 

9.     sc.pipeline().addLast(new DelimiterBasedFrameDecoder(1024, buf)); 

10.     //设置字符串类型的编码 将Buf类型转换成String类型 

11.     sc.pipeline().addLast(new StringDecoder()); 

12.     //加入自己定义的逻辑处理类 

13.     sc.pipeline().addLast(new ServerHandler()); 

14.   } 

15. }); 
```



当 Channel 被创建时，它会被自动地分配到它专属的 ChannelPipeline;一个Channel包含一个 ChannelPipeline;一个 pipeline 上可以有多个 ChannelHandler。

![img](http://pcc.huitogo.club/589b75746617430f80ed22e512ef8502)



我们可以在 ChannelPipeline 上通过 addLast() 方法添加一个或者多个ChannelHandler （一个数据或者事件可能会被多个 Handler 处理） 。当一个 ChannelHandler 处理完之后就将数据交给下一个 ChannelHandler 。



当 ChannelHandler 被添加到的 ChannelPipeline 它得到一个 ChannelHandlerContext，它代表一个 ChannelHandler 和 ChannelPipeline 之间的“绑定”。 ChannelPipeline 通过 ChannelHandlerContext来间接管理 ChannelHandler 。



#### 6. ChannelFuture（操作执行结果）

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