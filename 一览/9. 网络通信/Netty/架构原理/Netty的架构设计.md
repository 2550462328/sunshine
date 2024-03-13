#### 1. 逻辑架构

![Netty 逻辑架构图](https://pcc.huitogo.club/z0/85d98e3cb0e6e39f80d02234e039a4dd)

注意：这个图是自下向上看。

1. **Reactor 通信调度层**：由一系列辅助类组成，包括 Reactor 线程 NioEventLoop 及其父类，NioSocketChannel 和 NioServerSocketChannel 等等。该层的职责就是监听网络的读写和连接操作，负责将网络层的数据读到内存缓冲区，然后触发各自网络事件，例如连接创建、连接激活、读事件、写事件等。将这些事件触发到 pipeline 中，由 pipeline 管理的职责链来进行后续的处理。
2. **职责链 ChannelPipeline**：负责事件在职责链中的有序传播，以及负责动态地编排职责链。职责链可以选择监听和处理自己关心的事件，拦截处理和向后传播事件。
3. **业务逻辑编排层**：业务逻辑编排层通常有两类，一类是纯粹的业务逻辑编排，一类是应用层协议插件，用于特定协议相关的会话和链路管理。由于应用层协议栈往往是开发一次到处运行，并且变动较小，故而将应用协议到 POJO 的转变和上层业务放到不同的 ChannelHandler 中，就可以实现协议层和业务逻辑层的隔离，实现架构层面的分层隔离。



#### 2. 网络架构

![img](http://pcc.huitogo.club/02717d6156a6f62ad9ede8fdd6385225)



Nelty 的 整体架构分为 Transport Services ( 传输服务层)、 Protocol Support (传输协议层)和 Core (核心层)3 部分 。



#### 1. Transport Services

Transport Services 指传输 服务层 ， 主要定义了数据的传输和通信方式 ， 包括 Socket And Datagram ( Socket 协议和数据包) 、 HTTP Tunnel ( HTTP 隧道)、 In -VM Pipe (本地 传输管道)。

基于 Socket 的协议有 TCP 、 UDP 等 。 其中， TCP 基于报文应答机制实现，主要用于 可靠数据的传输 ， 比如移动设备状态信息的传输 。 UDP 发出 的数据不需要对方应答，主 要用 于对数据安全性要求不是很高但是对数据传输吞吐量要求较高的场景，比如实时视 频流传输 。 HTTP Tunnel 定义了 HTTP 的传输方式。 In-VM Pipe 定义了本地数据的传输方式 。



#### 2. Protocol Support

Protocol Support 指传输协议层 ， 主要定义数据传输过程中的服务类型、数据安全 、 数据压缩 等。 具体包括 HTTP And WebSockel ( HTTP 和 WebSocket 服务)、 SSL And StartTLS ( SSL 和 StartTLS 协议)、 z lib/gzip Compression ( zlib/gzip 压缩算法)、 Large File Transfer (大文件传输)、 Google ProtoBuf ( Google ProtoBuf 格式)、 RTSP (实时流传 输协议)、 Legacy Text And Binary Protocols (传统 TXT 和二进制数据)



#### 3. Core

Netty 的 Core (核心层)封装了 Netty 框架的核心服务和 API ，具体包括 Extensible Even t Model (可扩展事件模型)、 Universal Communication API (通用通信协议 API)、 Zero-Copy-Capable Rich Byte Buffer (零拷贝字节缓冲区)等。可扩展事件模型为 Netty 灵活的事件通信提供了基础；通用通信协议 API 为上层提供了统一的 API 访问人口，提 高了 Netty 框架的易用性，零拷贝字节缓冲区为数据的快速读取和写入提供了保障。