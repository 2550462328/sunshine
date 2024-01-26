#### 1. 为什么使用Selector？

只需要更少的线程来处理通道。事实上，可以只用一个线程处理所有的通道。对于操作系统来说，线程之间上下文切换的开销很大，而且每个线程都要占用系统的一些资源（如内存）。因此，使用的线程越少越好。



#### 2. Selector工作流程

使用selector操作多个通道获取对应事件的过程如下：

##### 2.1 创建selector

通过调用Selector.open()方法创建一个Selector，如下：

```
// 创建Selector

Selector selector = Selector.open();
```



##### 2.2 注册通道

```
channel.configureBlocking(false);  

SelectionKey key = channel.register(selector,Selectionkey.OP_READ);  
```

首先需要设置通道阻塞模式为非阻塞，所以不适用于FileChannel，所有的套接字传输的都可以注册。

第二个参数是监听的通道事件，有以下几个选项

Connect、Accept、Read、Write

监听多个使用“位或”操作符，如下：

```
int interestSet = SelectionKey.OP_READ | SelectionKey.OP_WRITE;
```

通过SelectionKey可以获取所有监听事件、准备就绪的事件、绑定的通道和Selector。



##### 2.3 获取激活的通道个数

```
int readyChannels = selector.select();
```

这里select()有重载方法

```
不带参数的select()，会一直阻塞直到有一个激活的通道，可以调用Selector.wakeup()来停止它，如果调用Selector.wakeup()的时候没有阻塞的select()，那么从现在开始的下一个select()会直接“醒来”。

select(long timeout),定义了超时时间的select，顾名思义，最大会阻塞到timeout。

selectNow()，这是一种非阻塞的select，有一个激活的通道就会返回，所以返回的激活通道数不是0就是1。
```



##### 2.4 获取激活的通道

```
// 获取激活的通道

Set selectedKeys = selector.selectedKeys();
```

然后可以进行遍历，获取selectedKey，判断激活事件类型，获取到Channel，进行逻辑处理

记得要**手动移除selectedKey**，因为Selector不会自己从已选择键集中移除SelectionKey实例。



##### 2.5 关闭通道

用完Selector后调用其close()方法会关闭该Selector，且使注册到该Selector上的所有SelectionKey实例无效。通道本身并不会关闭。



#### 3. Select的唤醒

select是会阻塞线程的，直到有数据可处理，怎么提前唤醒它呢？

```
// select唤醒

Selector.wakeup()
```



需要注意的有

1）两个select之间多次wakeup等价于调用一次wakeup。

2）wakeup调用时如果当前没有调用select方法，则生效于下一次。



为什么要唤醒它？

- 注册了新的channel或者事件。
- channel关闭，取消注册。
- 优先级更高的事件触发（如定时器事件），希望及时处理。



唤醒原理是什么？

wakeup往管道或者连接写入一个字节，阻塞的select因为有I/O事件就绪，立即返回。



#### 4. Selector使用实例

关于Selector和Channel、Buffer的简单实例如下：

```
1.  public static void main(String[] args) throws Exception {  

2.      // 看看有没有的新的客户端连接过来  

3.     Selector serverSelector = Selector.open();  

4.     // 看看连接的客户端有没有新的数据产生  

5.      Selector clientSelector = Selector.open();  

6.      new Thread(() -> {  

7.          try {  

8.              // 注册服务端 监听连接的客户端  

9.              ServerSocketChannel socketChannel = ServerSocketChannel.open();  

10.             socketChannel.configureBlocking(false);  

11.             socketChannel.bind(new InetSocketAddress(3333));  

12.             socketChannel.register(serverSelector, SelectionKey.OP_ACCEPT);  

13.             //自循环监听注册过来的客户端连接  

14.             while(true){  

15.                 // 这里1是1ms，超时时间  

16.                 if(serverSelector.select(1) > 0){  

17.                     Set<SelectionKey> selectionKeys =  serverSelector.selectedKeys();  

18.                     java.util.Iterator<SelectionKey> keyIterator = selectionKeys.iterator();  

19.                     while(keyIterator.hasNext()){  

20.                         try {  

21.                             SelectionKey selectionKey = keyIterator.next();  

22.                             // 这里发现到有新的客户端连接后不创建新的线程，而是注入到clientSelecotr中去，让clientSelector去监听数据  

23.                             SocketChannel clientChannel  = ((ServerSocketChannel) selectionKey.channel()).accept();  

24.                             clientChannel.configureBlocking(false);  

25.                             clientChannel.register(clientSelector,SelectionKey.OP_READ);  

26.                         } finally {  

27.                             // 移除当前已处理的客户端连接  

28.                             keyIterator.remove();  

29.                         }  

30.                     }  

31.                 }  

32.             }  

33.         } catch (IOException e) {  

34.            //Ingore  

35.         }  

36.     });  


37.     new Thread(()->{  

38.         // 自循环监听客户端传输过来的数据  

39.         try {  

40.             while(true){  

41.                 if(clientSelector.select(1) > 0){  

42.                     Set<SelectionKey> selectionKeys = clientSelector.selectedKeys();  

43.                     Iterator<SelectionKey> keyIterator = selectionKeys.iterator();  

44.                     while(keyIterator.hasNext()){  

45.                         SelectionKey clientKey = keyIterator.next();  

46.                         // 有没有数据可读  

47.                         if(clientKey.isReadable()){  

48.                             try {  

49.                                 SocketChannel socketChannel = (SocketChannel) clientKey.channel();  

50.                                 ByteBuffer byteBuffer = ByteBuffer.allocate(1024);  

51.                                 socketChannel.read(byteBuffer);  

52.                                 byteBuffer.flip();  

53.                                 System.out.println(Charset.defaultCharset().newDecoder().decode(byteBuffer).toString());  

54.                             } finally {  

55.                                 // 移除当前已处理的客户端读取数据  

56.                                 keyIterator.remove();  

57.                                 clientKey.interestOps(SelectionKey.OP_READ);  

58.                             }  

59.                         }  

60.                     }  

61.                 }  

62.             }  

63.         } catch (IOException e) {  

64.             //Ingore  

65.         }  

66.     });  

67. } 
```

可以看得出就算是一个简单的示例（没有考虑其他情况）代码已经如此复杂了。

所以这里推荐直接使用netty框架，现成的轮子