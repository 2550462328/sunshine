#### 1. Reactor模式

NIO在等待就绪中是异步的，这个在实际操作中如果一个连接不能读写（socket.read()返回0或者socket.write()返回0），我们可以把这件事件记下来，记录的方式通常是在Selector上注册标记位，然后切换到其它就绪的连接（channel）继续进行读写。

对于NIO需要记录的事件有：**读就绪**、**写就绪**和**新连接到来**

在记录事件之前我们要先**注册这几个事件到来的对应的处理器**。然后单线程去死循环选择就绪的事件（在java api上调用的是**select**方法，在linux上2.6之前是select、poll，2.6之后是**epoll**，Windows是IOCP），注意**select方法是阻塞的**，新事件到来的时候，会在selector上注册标记位，标志可读、可写或者有连接到来。通常**连接的处理和读写的处理通常可以选择分开**，这样对于海量连接的注册和读写就可以分发。

![img](http://pcc.huitogo.club/f995cacd9b39623478d80ced1c590fae)



这种事件处理器模式叫Reactor模式，我们总结下Reactor 模型的**核心思想**：

将关注的 I/O 事件注册到多路复用器上，一旦有 I/O 事件触发，将事件分发到事件处理器中，执行就绪 I/O 事件对应的处理函数中。模型中有三个重要的组件：

- 多路复用器：由操作系统提供接口，Linux 提供的 I/O 复用接口有select、poll、epoll 。
- 事件分离器：将多路复用器返回的就绪事件分发到事件处理器中。
- 事件处理器：处理就绪事件处理函数。



对应Reactor 有 3 种模型实现：

1. 单 Reactor 单线程模型
2. 单 Reactor 多线程模型
3. 多 Reactor 多线程模型



##### 1.1 单 Reactor 单线程模型

![单 Reactor 单线程模型](http://static.iocoder.cn/images/Netty/2018_05_01/01.png)

Reactor 示例代码如下：

```
/**
* 等待事件到来，分发事件处理
*/
class Reactor implements Runnable {

  private Reactor() throws Exception {
      SelectionKey sk = serverSocket.register(selector, SelectionKey.OP_ACCEPT);
      // attach Acceptor 处理新连接
      sk.attach(new Acceptor());
  }

  @Override
  public void run() {
      try {
          while (!Thread.interrupted()) {
              selector.select();
              Set selected = selector.selectedKeys();
              Iterator it = selected.iterator();
              while (it.hasNext()) {
                  it.remove();
                  //分发事件处理
                  dispatch((SelectionKey) (it.next()));
              }
          }
      } catch (IOException ex) {
          //do something
      }
  }

  void dispatch(SelectionKey k) {
      // 若是连接事件获取是acceptor
      // 若是IO读写事件获取是handler
      Runnable runnable = (Runnable) (k.attachment());
      if (runnable != null) {
          runnable.run();
      }
  }
}
```

这是最基础的单 Reactor 单线程模型。

Reactor 线程，负责多路分离套接字。

- 有新连接到来触发 `OP_ACCEPT` 事件之后， 交由 Acceptor 进行处理。
- 有 IO 读写事件之后，交给 Handler 处理。

Acceptor 主要任务是构造 Handler 。

- 在获取到 Client 相关的 SocketChannel 之后，绑定到相应的 Handler 上。
- 对应的 SocketChannel 有读写事件之后，基于 Reactor 分发，Handler 就可以处理了。

**注意，所有的 IO 事件都绑定到 Selector 上，由 Reactor 统一分发**。



该模型适用于处理器链中业务处理组件能快速完成的场景。不过，这种单线程模型不能充分利用多核资源，**所以实际使用的不多**。



##### 1.2 单 Reactor 多线程模型

![单 Reactor 多线程模型](http://static.iocoder.cn/images/Netty/2018_05_01/02.png)

相对于第一种单线程的模式来说，在处理业务逻辑，也就是获取到 IO 的读写事件之后，交由线程池来处理，这样可以减小主 Reactor 的性能开销，从而更专注的做事件分发工作了，从而提升整个应用的吞吐。



MultiThreadHandler 示例代码如下：

```
/**
* 多线程处理读写业务逻辑
*/
class MultiThreadHandler implements Runnable {
  public static final int READING = 0, WRITING = 1;
  int state;
  final SocketChannel socket;
  final SelectionKey sk;

  //多线程处理业务逻辑
  ExecutorService executorService = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());


  public MultiThreadHandler(SocketChannel socket, Selector sl) throws Exception {
      this.state = READING;
      this.socket = socket;
      sk = socket.register(selector, SelectionKey.OP_READ);
      sk.attach(this);
      socket.configureBlocking(false);
  }

  @Override
  public void run() {
      if (state == READING) {
          read();
      } else if (state == WRITING) {
          write();
      }
  }

  private void read() {
      //任务异步处理
      executorService.submit(() -> process());

      //下一步处理写事件
      sk.interestOps(SelectionKey.OP_WRITE);
      this.state = WRITING;
  }

  private void write() {
      //任务异步处理
      executorService.submit(() -> process());

      //下一步处理读事件
      sk.interestOps(SelectionKey.OP_READ);
      this.state = READING;
  }

  /**
    * task 业务处理
    */
  public void process() {
      //do IO ,task,queue something
  }
}
```

在 `#read()` 和 `#write()` 方法中，提交 `executorService` 线程池，进行处理。



##### 1.3 多 Reactor 多线程模型

![多 Reactor 多线程模型](http://static.iocoder.cn/images/Netty/2018_05_01/03.png)

第三种模型比起第二种模型，是将 Reactor 分成两部分：

1. mainReactor 负责监听 ServerSocketChannel ，用来处理客户端新连接的建立，并将建立的客户端的 SocketChannel 指定注册给 subReactor 。
2. subReactor 维护自己的 Selector ，基于 mainReactor 建立的客户端的 SocketChannel 多路分离 IO 读写事件，读写网络数据。对于业务处理的功能，另外扔给 worker 线程池来完成。



MultiWorkThreadAcceptor 示例代码如下：

```
/**
* 多work 连接事件Acceptor,处理连接事件
*/
class MultiWorkThreadAcceptor implements Runnable {

  // cpu线程数相同多work线程
  int workCount = Runtime.getRuntime().availableProcessors();
  SubReactor[] workThreadHandlers = new SubReactor[workCount];
  volatile int nextHandler = 0;

  public MultiWorkThreadAcceptor() {
      this.init();
  }

  public void init() {
      nextHandler = 0;
      for (int i = 0; i < workThreadHandlers.length; i++) {
          try {
              workThreadHandlers[i] = new SubReactor();
          } catch (Exception e) {
          }
      }
  }

  @Override
  public void run() {
      try {
          SocketChannel c = serverSocket.accept();
          if (c != null) {// 注册读写
              synchronized (c) {
                  // 顺序获取SubReactor，然后注册channel 
                  SubReactor work = workThreadHandlers[nextHandler];
                  work.registerChannel(c);
                  nextHandler++;
                  if (nextHandler >= workThreadHandlers.length) {
                      nextHandler = 0;
                  }
              }
          }
      } catch (Exception e) {
      }
  }
}
```



SubReactor 示例代码如下：

```
/**
* 多work线程处理读写业务逻辑
*/
class SubReactor implements Runnable {
  final Selector mySelector;

  //多线程处理业务逻辑
  int workCount =Runtime.getRuntime().availableProcessors();
  ExecutorService executorService = Executors.newFixedThreadPool(workCount);


  public SubReactor() throws Exception {
      // 每个SubReactor 一个selector 
      this.mySelector = SelectorProvider.provider().openSelector();
  }

  /**
    * 注册chanel
    *
    * @param sc
    * @throws Exception
    */
  public void registerChannel(SocketChannel sc) throws Exception {
      sc.register(mySelector, SelectionKey.OP_READ | SelectionKey.OP_CONNECT);
  }

  @Override
  public void run() {
      while (true) {
          try {
          //每个SubReactor 自己做事件分派处理读写事件
              selector.select();
              Set<SelectionKey> keys = selector.selectedKeys();
              Iterator<SelectionKey> iterator = keys.iterator();
              while (iterator.hasNext()) {
                  SelectionKey key = iterator.next();
                  iterator.remove();
                  if (key.isReadable()) {
                      read();
                  } else if (key.isWritable()) {
                      write();
                  }
              }

          } catch (Exception e) {

          }
      }
  }

  private void read() {
      //任务异步处理
      executorService.submit(() -> process());
  }

  private void write() {
      //任务异步处理
      executorService.submit(() -> process());
  }
  /**
    * task 业务处理
    */
  public void process() {
      //do IO ,task,queue something
  }
}
```

从代码中，我们可以看到：

1. mainReactor 主要用来处理网络 IO 连接建立操作，通常，mainReactor 只需要一个，因为它一个线程就可以处理。
2. subReactor 主要和建立起来的客户端的 SocketChannel 做数据交互和事件业务处理操作。通常，subReactor 的个数和 CPU 个数**相等**，每个 subReactor **独占**一个线程来处理。



此种模式中，每个模块的工作更加专一，耦合度更低，性能和稳定性也大大的提升，支持的可并发客户端数量可达到上百万级别。

关于此种模式的应用，目前有很多优秀的框架已经在应用，比如 Mina 和 Netty 等等。**上述中去掉线程池的第三种形式的变种，也是 Netty NIO 的默认模式**。



#### 2. Redis中的Reactor

redis为什么快的原因除了它是基于内存的之外，还有一个重要的原因它是使用单线程+多路复用IO模型的。

![img](http://pcc.huitogo.club/fd9e103319d9423ecc7d60c201753670)



它内部的实现使用单线程＋队列，把请求数据缓冲。然后pipeline发送，返回future，然后channel可读时，直接在队列中把future取回来，done()就可以了。



伪代码如下：

```
1.  class RedisClient Implements ChannelHandler{  

2.   private BlockingQueue CmdQueue;  

3.   private EventLoop eventLoop;  

4.   private Channel channel;  

5.   class Cmd{  

6.    String cmd;  

7.    Future result;  

8.   }  

9.   public Future get(String key){  

10.    Cmd cmd= new Cmd(key);  

11.    queue.offer(cmd);  

12.    eventLoop.submit(new Runnable(){  

13.         List list = new ArrayList();  

14.         queue.drainTo(list);  

15.         if(channel.isWritable()){  

16.          channel.writeAndFlush(list);  

17.         }  

18.    });  

19. }  

20.  public void ChannelReadFinish(Channel channel，Buffer Buffer){  

21.     List result = handleBuffer();//处理数据  

22.     //从cmdQueue取出future，并设值，future.done();  

23. }  

24.  public void ChannelWritable(Channel channel){  

25.    channel.flush();  

26. }  

27. } 
```



#### 3. Proactor模式

之前我们讲了NIO事件处理器是基于Reactor模式的，那么Proactor模式是什么呢？

再回顾一下Reactor模式中读的流程：

- 注册读就绪事件和相应的事件处理器。
- 事件分发器等待事件。
- 事件到来，激活分发器，分发器调用事件对应的处理器。
- 事件处理器完成实际的读操作，处理读到的数据，注册新的事件，然后返还控制权。



那么在Proactor模式下读的流程：

- 处理器发起异步读操作（注意：操作系统必须支持异步IO）。在这种情况下，处理器无视IO就绪事件，它关注的是完成事件。
- 事件分发器等待操作完成事件。
- 在分发器等待过程中，操作系统利用并行的内核线程执行实际的读操作，并将结果数据存入用户自定义缓冲区，最后通知事件分发器读操作完成。
- 事件分发器呼唤处理器。
- 事件处理器处理用户自定义缓冲区中的数据，然后启动一个新的异步操作，并将控制权返回事件分发器。



#### 4. Reactor和Proactor

可以看出，两个模式的相同点，都是对某个I/O事件的事件通知（即告诉某个模块，这个I/O操作可以进行或已经完成)。在结构上，两者也有相同点：事件分发器负责提交IO操作（异步)、查询设备是否可操作（同步)，然后当条件满足时，就回调handler；不同点在于，异步情况下（Proactor)，当回调handler时，表示I/O操作已经完成；同步情况下（Reactor)，回调handler时，表示I/O设备可以进行某个操作（can read 或 can write)。

所以**Reactor模式是同步的，它回调给用户的时候是通知可以进行读写了，而Proactor是异步的，它回调给用户的时候是通知读写已经完成了。**



那么我们可以将Reactor模式变成Proactor模式吗？伪异步？

我们分析Reactor和Proactor的时候发现，两者区别主要是Reactor只是通知用户可以进行读写了，而并没有帮用户去做这件事，导致接下来的读写操作还是同步进行的。

所以从这点下手，如果内核通知事件处理器可以读写的时候，那我们就直接读写，不用去通知用户让用户来读写，最后通知用户读写完成，调用用户定义的读写完成的业务逻辑。



伪代码如下，在原有代码下改装

```
1.  interface ChannelHandler{  

2.        void channelReadComplate(Channel channel，byte[] data);  

3.        void channelWritable(Channel channel);  

4.     }  

5.     class Channel{  

6.       Socket socket;  

7.       Event event;//读，写或者连接  

8.     }  

9.     //IO线程主循环：  

10.    class IoThread extends Thread{  

11.    public void run(){  

12.    Channel channel;  

13.    while(channel=Selector.select()){//选择就绪的事件和对应的连接  

14.       if(channel.event==accept){  

15.          registerNewChannelHandler(channel);//如果是新连接，则注册一个新的读写处理器  

16.          Selector.interested(read);  

17.       }  

18.       if(channel.event==write){  

19.          getChannelHandler(channel).channelWritable(channel);//如果可以写，则执行写事件  

20.       }  

21.       if(channel.event==read){  

22.           byte[] data = channel.read();  //这里直接读

23.           if(channel.read()==0)//没有读到数据，表示本次数据读完了  

24.           {  

25.           getChannelHandler(channel).channelReadComplate(channel，data;//处理读完成事件  

26.           }  

27.           if(过载保护){  

28.           Selector.interested(read);  

29.           }  

30.       }  

31.      }  

32.     }  

33.    Map<Channel，ChannelHandler> handlerMap;//所有channel的对应事件处理器  

34.    }  
```



现在我们分析一下在Reactor模式下和伪Proactor模式下读的流程

在Reacotr模式下

步骤1：等待事件到来（Reactor负责）。

步骤2：将读就绪事件分发给用户定义的处理器（Reactor负责）。

步骤3：读数据（用户处理器负责）。

步骤4：处理数据（用户处理器负责）。



在伪Proactor模式下

步骤1：等待事件到来（Proactor负责）。

步骤2：得到读就绪事件，执行读数据（现在由Proactor负责）。

步骤3：将读完成事件分发给用户处理器（Proactor负责）。

步骤4：处理数据（用户处理器负责）。
