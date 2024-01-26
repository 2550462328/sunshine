#### 1. Reactor模式

NIO在等待就绪中是异步的，这个在实际操作中如果一个连接不能读写（socket.read()返回0或者socket.write()返回0），我们可以把这件事件记下来，记录的方式通常是在Selector上注册标记位，然后切换到其它就绪的连接（channel）继续进行读写。

对于NIO需要记录的事件有：**读就绪**、**写就绪**和**新连接到来**

在记录事件之前我们要先**注册这几个事件到来的对应的处理器**。然后单线程去死循环选择就绪的事件（在java api上调用的是**select**方法，在linux上2.6之前是select、poll，2.6之后是**epoll**，Windows是IOCP），注意**select方法是阻塞的**，新事件到来的时候，会在selector上注册标记位，标志可读、可写或者有连接到来。通常**连接的处理和读写的处理通常可以选择分开**，这样对于海量连接的注册和读写就可以分发。



![img](http://pcc.huitogo.club/f995cacd9b39623478d80ced1c590fae)



对于以上过程，程序的伪代码是

```
1.  interface ChannelHandler{  

2.     void channelReadable(Channel channel);  

3.     void channelWritable(Channel channel);  

4.  }  


5.  class Channel{  

6.    Socket socket;  

7.    Event event;//读，写或者连接  

8.  }  


9.  //IO线程主循环:  

10. class IoThread extends Thread{  

11. public void run(){  

12. Channel channel;  

13. while(channel=Selector.select()){//选择就绪的事件和对应的连接，会阻塞  

14.    if(channel.event==accept){  

15.       registerNewChannelHandler(channel);//如果是新连接，则注册一个新的读写处理器  

16.    }  

17.    if(channel.event==write){  

18.       getChannelHandler(channel).channelWritable(channel);//如果可以写，则执行写事件  

19.    }  

20.    if(channel.event==read){  

21.        getChannelHandler(channel).channelReadable(channel);//如果可以读，则执行读事件  

22.    }  

23.  }  

24. }  

25. Map<Channel，ChannelHandler> handlerMap;//所有channel的对应事件处理器  
```



这种事件处理器模式叫Reactor模式，以上也是Reactor模式最简单的实现，我们分析一下这种模式下我们实现中需要使用线程的地方

1. 事件分发器，单线程选择就绪的事件。

2. I/O处理器，包括connect、read、write等，这种纯CPU操作，一般开启CPU核心个线程就可以。

3. 业务线程，在处理完I/O后，业务一般还会有自己的业务逻辑，有的还会有其他的阻塞I/O，如DB操作，RPC等。**只要有阻塞，就需要单独的线程**。




需要注意的一点是，在Linux下，对于**同一个Channel的select不能被并发调用，**因此，如果有多个I/O线程，必须保证：一个socket只能属于一个IOThread，而一个IOThread可以管理多个socket。



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



可以看出，两个模式的相同点，都是对某个I/O事件的事件通知（即告诉某个模块，这个I/O操作可以进行或已经完成)。在结构上，两者也有相同点：事件分发器负责提交IO操作（异步)、查询设备是否可操作（同步)，然后当条件满足时，就回调handler；不同点在于，异步情况下（Proactor)，当回调handler时，表示I/O操作已经完成；同步情况下（Reactor)，回调handler时，表示I/O设备可以进行某个操作（can read 或 can write)。

所以**Reactor模式是异步的，它回调给用户的时候是通知可以进行读写了，而Proactor是同步的，它回调给用户的时候是通知读写已经完成了。**



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
