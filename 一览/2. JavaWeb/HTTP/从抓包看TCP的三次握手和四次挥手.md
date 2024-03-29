通过抓包工具抓取浏览器对服务器的一次访问请求

![img](http://pcc.huitogo.club/28eb9c3928b71e804d1820c2932ef854)



可以看出请求Http协议之前进行了Tcp三次握手，Http协议之后进行了Tcp的四次挥手。

了解一下tcp握手和挥手时发送的内容（解析后的），即上图中Info里面的数据

- 54627->8662  端口号：源端口 --> 目标端口

- [SYN] : 同步握手信号

- [ACK] : 确认接收信号

- [FIN]：同步挥手信号

- Seq : 消息编号

- Win: TCP 窗口大小

  是指 TCP传输能接受的最大字节数，这个可以进行动态调节，也就是 TCP的滑动窗口，通过动态调整窗口大小，来控制发送数据的速率。上图中占用 2个字节，也就是 16位，那么可以支持的最大数就是 2^16=65536，所以默认情况下 TCP头部标记能支持的最大窗口数是 65536字节，也就是 64KB。

- Len: 消息长度

  就是指数据报文段，因为整个 TCP报文 = Header + packSize,所以这个消息长度就是指要传送的数据包总共长度，在本次分析中也就是 HTTP报文的大小。

- Mss: 最大报文段长度

  这个就是规定最大的能传输报文的长度，为了达到最佳的传输效能， TCP 协议在建立连接的时候通常要协商双方的 MSS 值，这个值 TCP 协议在实现的时候往往用 MTU 值代替（需要减去 IP数据包包头的大小 20Bytes和 TCP数据段的包头 20Bytes）所以一般 MSS 值 1460，这也和我们抓包图中的值一致。

- Ws: 窗口缩放调整因子

  默认情况下， TCP 窗口大小最大只能支持 64KB的缓冲数据，在今天这个高速上网时代，这个大小肯定不满足条件了，所以，为了能够支持更多的缓冲数据 RFC 1323中就规定了 TCP 的扩展选项，其中窗口缩放调整因子就是其中之一，这个是如何起作用的呢？首先说明，这个参数是在 [SYN] 同步阶段进行协商的，我们结合上面抓包数据分析下。我们看到第一次请求协商的结果是 WS=256,然后再 ACK 阶段扩展因子生效，调整了窗口大小。生效的抓包如下：

![img](http://pcc.huitogo.club/0b7bfba54161bf6761909405b6bff6d7)

点击查看报文信息：

![img](http://pcc.huitogo.club/7f577f2594b785f05c0d05a70e54e140)

我们发现，实际请求声明的窗口是 1024， WS扩展因子是 256，最终计算的窗口大小是 262144，所以我们知道了，这个扩展因子的作用就是，**用原窗口大小乘以扩展因子，得到最终的窗口大小**，也就是 1024*256=262144。

- SACK_PERM : SACK选项，这里等于1表示开启 SACK。

我们知道 TCP 传输有包的确认机制，默认情况下，接受端接受到一个包后，发送 ACK 确认，但是，默认只支持顺序的确认，也就是说，发送 A, B, C 个包，如果我收到了 A, C的包， B没有收到，那么对于 C，这个包我是不会确认的，需要等 B这个包收到后再确认，那么 TCP有超时重传机制，如果一个包很久没有确认，就会当它丢失了，进行重传，这样会造成很多多余的包重传，浪费传输空间。为了解决这个问题， SACK就提出了选择性确认机制，启用 SACK 后，接受端会确认所有收到的包，这样发送端就只用重传真正丢失的包了



在Tcp三次握手后，可以进行Http通信，在抓包图中：

![img](http://pcc.huitogo.club/3591943989beef04fb3d96dc989a620b)

可以看到请求和返回的信息



在一次Http通信后，就要进行Tcp的4次挥手

![img](http://pcc.huitogo.club/d2c3672a4e247c824771be6b9f03a2f3)



这里是有完整的4次挥手，**为什么有些情况是会出现3次挥手的情况？**

分析一下原因：

4次挥手过程如下图：

![img](http://pcc.huitogo.club/85f5ef032ee65e31043363270d260d49)



挥手流程是这样的：

1. 客户端发起一个断开请求，进入 FIN-WAIT 状态
2. 服务端确认断开请求
3. 服务端立即发送一个断开请求，进入 CLOSE-WAIT 状态
4. 客户端确认服务端断开请求，进入 TIME-WAIT 状态

流程2和 流程3都是由服务端发起的，那么有没有**可能合并这两个请求**，一次发送给客户端？答案是可以。在 RFC 2581中的 4.2 节有提到， **ack可以延迟确认，只要求保证在 500ms之内保证确认包到达即可**。在这样的标准下， TCP确认是有可能进行合并延迟确认的



**是不是每次连接都会都会有三次握手？**

在 HTTP0.9 版本和 HTTP1.0 版本中，每次请求响应都是要三次握手的， 但是 HTTP1.0 开始尝试持续连接，也就是 Keep-Alive 参数，但是官方还没有正式支持，在 HTTP1.1协议中，官方默认就是支持 Keep-Alive 参数的，默认是持续连接。 



**Keep-Alive 的作用**主要有两点：

1） **检查死节点**

主要是为了让连接快速失败被发现，可以进行重新连接，比如 A 和 B 两端已经建立了连接， B节点因为 异常原因挂掉了，同时 A 节点并不知道，这时候有两种情况：

· 假设 B 节点还没有恢复，那么 B 节点不会回复 ACK， A节点就会一直重试，重试到一定次数才能知道 B 节点是死节点。

· B节点在 A发送数据之前重启成功了，这个时候 A节点发送数据， B节点并不会接受，而是会发送一个 RST 信号（在一个已关闭的 socket 上收到数据时，将发送 RST数据包，要求对端关闭异常连接且对端不需要回复 ACK）,然后 A 才知道 B 节点需要重连了。

以上两种情况，都会导致只有到发送数据的时候才知道对方已经出异常了。而 Keep-Alive 每隔一段时间就会发送心跳，就可以很快的知道服务端节点的情况。



2）**防止连接由于不活跃而断开**

我们知道，网络连接的建立和维持是消耗资源的，一个服务器上能建立的连接是有限的，所以像防火墙或者操作系统中会为了节省资源会释放掉不活跃的连接，而 Keep-Alive 每隔一段时间发送一个心跳包，就是告诉防火墙或者操作系统，我这个连接是活跃的，不要杀我。

通过下图可以看出建立连接一段时间后，客户端会定时发送心跳包保持keep-alive

![img](http://pcc.huitogo.club/b5005c17a26cbfff8e58558683f11f27)



**为什么有时候浏览器（客户端）会一开始会有两个端口发送请求？**

其实，这个和协议本身没有任何关系，一开始我们抓包截图是用的谷歌浏览器，会出现两个端口三次握手，后面用火狐浏览器只有一个端口三次握手，所以**这种情况的发生就是浏览器自身的实现**，谷歌浏览器为什么会这么实现，猜测是：**尽可能的保证HTTP访问的可用性，当某个端口不可用，可以立即切换到另外一个端口，完成HTTP的请求和响应**。