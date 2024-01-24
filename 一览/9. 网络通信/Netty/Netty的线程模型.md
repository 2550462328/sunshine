我们先看一下netty的工作机制

![img](http://pcc.huitogo.club/6536f59195a017c0eeb62d52adf1bf25)



Netty 的运行核心包含两个 NioEventLoopGroup 工作组， 一个是 BossGroup ，用于接 收客户瑞连接 、 接收客户端数据和进行数据转发；另一个是 WorkerGroup ，用于具体I/O事件的触发 和数据处理。



**BossGroup 的职责**

BossGroup 为一个事件循环组， 其中 包 含多个事 件 循环( NioEventLoop )，每个NioEventLoop 都包含一个 Selector 和 一个 TaskQueue (事件循环线程)。



每个 Boss NioEventLoop 循环都执行以下 3 个步骤。

1. 轮询监听 Accept 事件 。
2. 接收和处理 Accept 事件，包括和客户端建立连接并生成 NioSocketChannel ，将 NioSocketChannel 注册到某个 Worker NioEventLoop 的 Selector 上 。
3. 处理 runAIITasks 的任务。



**WorkerGroup 的职责**

WorkerGroup 为一个 事件循环组， 中包含 多个事 件循环 (NioEventLoop )。



每个 Worker NioEventLoop 循环都执行以下 3 个步骤 。

1. 轮询监听 NioSocketChannel 上的 I/O事件(I/O读写事件)。
2. 当 NioSocketChannel 有 I/O 事件触发时执行具体的 I/O 操作 。
3. 处理任务队列中的任务。



我们基于netty的工作机制来看它的线程模型



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