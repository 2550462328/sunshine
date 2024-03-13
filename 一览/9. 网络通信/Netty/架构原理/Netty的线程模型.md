我们先看一下netty的工作机制

![img](http://pcc.huitogo.club/6536f59195a017c0eeb62d52adf1bf25)



Netty 的运行核心包含两个 NioEventLoopGroup 工作组， 一个是 BossGroup ，用于接 收客户瑞连接 、 接收客户端数据和进行数据转发；另一个是 WorkerGroup ，用于具体I/O事件的触发 和数据处理。



**BossGroup 的职责**

BossGroup 为一个事件循环组， 其中 包 含多个事件循环( NioEventLoop )，每个NioEventLoop 都包含一个 Selector 和 一个 TaskQueue (事件循环线程)。



每个 Boss NioEventLoop 循环都执行以下 3 个步骤。

1. 轮询监听 Accept 事件 。
2. 接收和处理 Accept 事件，包括和客户端建立连接并生成 NioSocketChannel ，将 NioSocketChannel 注册到某个 Worker NioEventLoop 的 Selector 上 。
3. 处理 runAIITasks 的任务。



**WorkerGroup 的职责**

WorkerGroup 为一个事件循环组， 中包含多个事件循环 (NioEventLoop )。



每个 Worker NioEventLoop 循环都执行以下 3 个步骤 。

1. 轮询监听 NioSocketChannel 上的 I/O事件(I/O读写事件)。
2. 当 NioSocketChannel 有 I/O 事件触发时执行具体的 I/O 操作 。
3. 处理任务队列中的任务。



我们基于netty的工作机制来看它有哪些Reactor线程模型



#### 1. 单线程模型

所有的IO操作都由同一个NIO线程处理。

![img](http://pcc.huitogo.club/ef6c14a20bcb2f82b17e402933520696)



单线程 Reactor 的优点是对系统资源消耗特别小，但是，没办法支撑大量请求的应用场景并且处理请求的时间可能非常慢，毕竟只有一个线程在工作嘛！所以，一般实际项目中不会使用单线程Reactor 。



#### 2. 多线程模型

一个线程负责接受请求,一组NIO线程处理IO操作。

![img](http://pcc.huitogo.club/4f6fa7018158564a0a3753334dba7be8)



大部分场景下多线程Reactor模型是没有问题的，但是在一些并发连接数比较多（如百万并发）的场景下，一个线程负责接受客户端请求就存在性能问题了。



#### 3. 主从多线程

一组NIO线程负责接受请求，一组NIO线程处理IO操作。

![img](http://pcc.huitogo.club/9b42091ebe311c5bee3cb005dd9eb359)



关于配置线程模型，就是通过EventLoopGroup了

```
1. EventLoopGroup pGroup = new NioEventLoopGroup(); //用来处理服务器端接受客户端连接的 

2. EventLoopGroup cGroup = new NioEventLoopGroup(); //用来进行网络读写的 
```

默认是**avaliableCPUNums \* 2**



那么NIO客户端和NIO服务端分别支持哪些模式？

#### 4. Netty NIO 客户端

我们来看看 Netty NIO 客户端的示例代码中，和 EventLoop 相关的代码：

```
// 创建一个 EventLoopGroup 对象
EventLoopGroup group = new NioEventLoopGroup();
// 创建 Bootstrap 对象
Bootstrap b = new Bootstrap();
// 设置使用的 EventLoopGroup
b.group(group);
```

- 对于 Netty NIO 客户端来说，仅创建一个 EventLoopGroup 。
- 一个 EventLoop 可以对应一个 Reactor 。因为 EventLoopGroup 是 EventLoop 的分组，所以对等理解，EventLoopGroup 是**一种** Reactor 的分组。
- 一个 Bootstrap 的启动，只能发起对一个远程的地址。所以只会使用一个 NIO Selector ，也就是说仅使用**一个** Reactor 。即使，我们在声明使用一个 EventLoopGroup ，该 EventLoopGroup 也只会分配一个 EventLoop 对 IO 事件进行处理。
- 因为 Reactor 模型主要使用服务端的开发中，如果套用在 Netty NIO 客户端中，到底使用了哪一种模式呢？
  - 如果只有一个业务线程使用 Netty NIO 客户端，那么可以认为是【单 Reactor **单**线程模型】。
  - 如果有**多个**业务线程使用 Netty NIO 客户端，那么可以认为是【单 Reactor **多**线程模型】。
- 那么 Netty NIO 客户端是否能够使用【多 Reactor 多线程模型】呢？创建多个 Netty NIO 客户端，连接同一个服务端。那么多个 Netty 客户端就可以认为符合多 Reactor 多线程模型了。
  - 一般情况下，我们不会这么干。
  - 当然，实际也有这样的示例。例如 Dubbo 或 Motan 这两个 RPC 框架，支持通过配置，同一个 Consumer 对同一个 Provider 实例同时建立多个客户端连接。



#### 5. Netty NIO 服务端

我们来看看 Netty NIO 服务端的示例代码中，和 EventLoop 相关的代码：

```
// 创建两个 EventLoopGroup 对象
EventLoopGroup bossGroup = new NioEventLoopGroup(1); // 创建 boss 线程组 用于服务端接受客户端的连接
EventLoopGroup workerGroup = new NioEventLoopGroup(); // 创建 worker 线程组 用于进行 SocketChannel 的数据读写
// 创建 ServerBootstrap 对象
ServerBootstrap b = new ServerBootstrap();
// 设置使用的 EventLoopGroup
b.group(bossGroup, workerGroup);
```

- 对于 Netty NIO 服务端来说，创建两个 EventLoopGroup 。
  - `bossGroup` 对应 Reactor 模式的 mainReactor ，用于服务端接受客户端的连接。比较特殊的是，传入了方法参数 `nThreads = 1` ，表示只使用一个 EventLoop ，即只使用一个 Reactor 。这个也符合我们上面提到的，“**通常，mainReactor 只需要一个，因为它一个线程就可以处理**”。
  - `workerGroup` 对应 Reactor 模式的 subReactor ，用于进行 SocketChannel 的数据读写。对于 EventLoopGroup ，如果未传递方法参数 `nThreads` ，表示使用 CPU 个数 Reactor 。这个也符合我们上面提到的，“**通常，subReactor 的个数和 CPU 个数相等，每个 subReactor 独占一个线程来处理**”。
- 因为使用两个 EventLoopGroup ，所以符合【多 Reactor 多线程模型】的多 Reactor 的要求。实际在使用时，`workerGroup` 在读完数据时，具体的业务逻辑处理，我们会提交到**专门的业务逻辑线程池**，例如在 Dubbo 或 Motan 这两个 RPC 框架中。这样一来，就完全符合【多 Reactor 多线程模型】。
- 那么可能有胖友可能和我有一样的疑问，`bossGroup` 如果配置多个线程，是否可以使用**多个 mainReactor** 呢？我们来分析一波，一个 Netty NIO 服务端**同一时间**，只能 bind 一个端口，那么只能使用一个 Selector 处理客户端连接事件。又因为，Selector 操作是非线程安全的，所以无法在多个 EventLoop ( 多个线程 )中，同时操作。所以这样就导致，即使 `bossGroup` 配置多个线程，实际能够使用的也就是一个线程。
- 那么如果一定一定一定要多个 mainReactor 呢？创建多个 Netty NIO 服务端，并绑定多个端口。