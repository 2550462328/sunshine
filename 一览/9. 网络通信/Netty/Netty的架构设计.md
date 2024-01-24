![img](http://pcc.huitogo.club/02717d6156a6f62ad9ede8fdd6385225)



Nelty 的 整体架构分为 Transport Services ( 传输服务层)、 Protocol Support (传输协议 层)和 Core (核心层)3 部分 。



#### 1. Transport Services

Transport Services 指传输 服务层 ， 主要定义了数据的传输和通信方式 ， 包括 Socket And Datagram ( Socket 协议和数据包) 、 HTTP Tunnel ( HTTP 隧道)、 In -VM Pipe (本地 传输管道)。

基于 Socket 的协议有 TCP 、 UDP 等 。 其中， TCP 基于报文应答机制实现，主要用于 可靠数据的传输 ， 比如移动设备状态信息的传输 。 UDP 发出 的数据不需要对方应答，主 要用 于对数据安全性要求不是很高但是对数据传输吞吐量要求较高的场景，比如实时视 频流传输 。 HTTP Tunnel 定义了 HTTP 的传输方式。 In-VM Pipe 定义了本地数据的传输方 式 。



#### 2. Protocol Support

Protocol Support 指传输协议层 ， 主要定义数据传输过程中的服务类型、数据安全 、 数据压缩 等。 具体包括 HTTP And WebSockel ( HTTP 和 WebSocket 服务)、 SSL And StartTLS ( SSL 和 StartTLS 协议)、 z lib/gzip Compression ( zlib/gzip 压缩算法)、 Large File Transfer (大文件传输)、 Google ProtoBuf ( Google ProtoBuf 格式)、 RTSP (实时流传 输协议)、 Legacy Text And Binary Protocols (传统 TXT 和二进制数据)



#### 3. Core

Netty 的 Core (核心层)封装了 Netty 框架的核心服务和 API ，具体包括 Extensible Even t Model (可扩展事件模型)、 Universal Communication API (通用通信协议 API)、 Zero-Copy-Capable Rich Byte Buffer (零拷贝字节缓冲区)等。可扩展事件模型为 Netty 灵活的事件通信提供了基础；通用通信协议 API 为上层提供了统一的 API 访问人口，提 高了 Netty 框架的易用性，零拷贝字节缓冲区为数据的快速读取和写入提供了保障。