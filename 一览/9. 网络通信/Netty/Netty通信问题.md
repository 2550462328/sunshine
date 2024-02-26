#### 1. 消息的粘包黏包问题

问题描述：TCP协议是流协议，服务器端发送数据 ABC DEF，客户端解析成AB DE F或者其他格式，如果不做处理的话这种情况很容易出现。



详细原因

1. 应用程序写入的字节大小大于套接字发送缓冲区的大小，会发生**拆包**现象。而应用程序写入数据小于套接字缓冲区大小，网卡将应用多次写入的数据发送到网络上，这将会发生**粘包**现象。
2. 待发送数据大于 MSS（最大报文长度），TCP 在传输前将进行**拆包**。
3. 以太网帧的 payload（净荷）大于 MTU（默认为 1500 字节）进行 IP 分片**拆包**。
4. 接收数据端的应用层没有及时读取接收缓冲区中的数据，将发生**粘包**。



解决方案

**1）使用 Netty 自带的解码器**

在 Netty 中，提供了多个 Decoder 解析类，如下：

- ① FixedLengthFrameDecoder ，基于**固定长度**消息进行粘包拆包处理的。
- ② LengthFieldBasedFrameDecoder ，基于**消息头指定消息长度**进行粘包拆包处理的。
- ③ LineBasedFrameDecoder ，基于**换行**来进行消息粘包拆包处理的。
- ④ DelimiterBasedFrameDecoder ，基于**指定消息边界方式**进行粘包拆包处理的。



实际上，上述四个 FrameDecoder 实现可以进行规整：

- ① 是 ② 的特例，**固定长度**是**消息头指定消息长度**的一种形式。
- ③ 是 ④ 的特例，**换行**是于**指定消息边界方式**的一种形式。



**2）自定义序列化编解码器**

在 Java 中自带的有实现 Serializable 接口来实现序列化，但由于它性能、安全性等原因一般情况下是不会被使用到的。

通常情况下，我们使用 Protostuff、Hessian2、json 序列方式比较多，另外还有一些序列化性能非常好的序列化方式也是很好的选择：

- 专门针对 Java 语言的：Kryo，FST 等等

- 跨语言的：Protostuff（基于 protobuf 发展而来），ProtoBuf，Thrift，Avro，MsgPack 等等




**3）推荐方案，将消息分成消息头和消息体，在消息头中包含消息长度的字段，然后进行业务逻辑的处理**



#### **2. Netty的通信方式**

通信方式大致分为三种：

1. 长连接不间断的进行连接，也就是服务端和客户端的通道一直处于开启状态

   推荐场景：服务器性能比较好，客户端连接数量比较少。

2. 一次性批量提交数据，采用短连接方式，也就是我们将数据保存在本地临时缓冲区或者临时表中，界当到达临时一次性批量提交，或者根据定时任务轮询提交

   推荐场景：对实时性要求不高。

3. 使用一种特殊的长连接，在指定的某一个时间，**服务器没有和客户端通信时断开连接，当客户端向服务端发送请求时再次建立连接**。这种场景需要考虑到



#### 3. Netty的长连接和短连接切换

**默认netty连接都是长连接**，如何切换成短连接呢？

在server端每次发完消息后关闭和客户端的连接，即在操作结尾加上addListener关闭对客户端的通道，但是客户端发送过来的消息还是会接收到。

注：writeAndFlush操作会把数据写到tcp流里面 并发送到通道里面，单纯的write并不会被服务端读取数据



#### 4. Netty怎么关闭空闲连接？

在 Netty 中，提供了 IdleStateHandler 类，正如其名，空闲状态处理器，用于检测连接的读写是否处于空闲状态。如果是，则会触发 IdleStateEvent 。



IdleStateHandler 目前提供三种类型的心跳检测，通过构造方法来设置。代码如下：

```
// IdleStateHandler.java
public IdleStateHandler(
        int readerIdleTimeSeconds,
        int writerIdleTimeSeconds,
        int allIdleTimeSeconds) {
    this(readerIdleTimeSeconds, writerIdleTimeSeconds, allIdleTimeSeconds,
         TimeUnit.SECONDS);
}
```

其中：

- `readerIdleTimeSeconds` 参数：为读超时时间，即测试端一定时间内未接受到被测试端消息。
- `writerIdleTimeSeconds` 参数：为写超时时间，即测试端一定时间内向被测试端发送消息。
- `allIdleTimeSeconds` 参数：为读或写超时时间。



#### 5. Netty如何重连？

###### 5.1 客户端

客户端，通过 IdleStateHandler 实现定时检测是否空闲，例如说 15 秒。

- 如果空闲，则向服务端发起心跳。
- 如果多次心跳失败，则关闭和服务端的连接，然后重新发起连接。



例如在NettyClient中设置 IdleStateHandler 和 ClientHeartbeatHandler：

```
// NettyHandler.java

.addLast("idleState", new IdleStateHandler(TaroConstants.TRANSPORT_CLIENT_IDLE, TaroConstants.TRANSPORT_CLIENT_IDLE, 0, TimeUnit.MILLISECONDS))
.addLast("heartbeat", new ClientHeartbeatHandler())
```

ClientHeartbeatHandler 中，碰到空闲，则发起心跳。不过，如何重连，暂时没有实现。需要考虑，重新发起连接可能会失败的情况。



###### 5.2 服务端

服务端，通过 IdleStateHandler 实现定时检测客户端是否空闲，例如说 90 秒。

- 如果检测到空闲，则关闭客户端。
- 注意，如果接收到客户端的心跳请求，要反馈一个心跳响应给客户端。通过这样的方式，使客户端知道自己心跳成功。



例如在NettyServer中设置 IdleStateHandler 和 ServerHeartbeatHandler：

```
// NettyServer.java

.addLast("idleState", new IdleStateHandler(0, 0, TaroConstants.TRANSPORT_SERVER_IDLE, TimeUnit.MILLISECONDS))
.addLast("heartbeat", new ServerHeartbeatHandler())
```

ServerHeartbeatHandler 中，检测到客户端空闲，则直接关闭连接。