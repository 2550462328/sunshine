Channel类似流，但是有所不同，如下：

- 既可以从通道中读取数据，又可以写数据到通道。但流的读写通常是单向的。

- 通道可以异步地读写。

- 通道中的数据总是要先读到一个Buffer，或者总是要从一个Buffer



### 1. FileChannel

我们无法直接打开一个FileChannel，需要通过使用一个InputStream、OutputStream或RandomAccessFile来获取一个FileChannel实例。

```
1.  RandomAccessFile randomAccessFile = new RandomAccessFile ("C:\\Users\\Dell\\Desktop\\2.txt", "rw");   

2.  //获取通道  

3.  FileChannel fileChannel = randomAccessFile.getChannel();  

4.  //创建字节缓冲区，capablity为12个字节  

5.  //gbk一个中文三个字节，utf8一个中文两个字节，其他都是一个字节  

6.  ByteBuffer byteBuffer = ByteBuffer.allocate(12);  

7.  //从通道中写字节到ByteBuffer  

8.  fileChannel.write(byteBuffer);  

9.  //获取ByteBuffer的limit  

10. int bytesRead = fileChannel.read(byteBuffer);  

11. while(bytesRead != -1) {  

12.     System.out.println("Read = " + bytesRead);  

13.     //从写模式切换到读模式，position归0，设置可读的limit  

14.     byteBuffer.flip();  

15.     System.out.println(new String(byteBuffer.array(), "gbk"));  

16.     while(byteBuffer.hasRemaining()) {  

17.         System.out.println(byteBuffer.get());  

18.     }  

19.     //并不是清空ByteBuffer，而是将读指针设置到ByteBuffer的limit  

20.     byteBuffer.clear();  

21.     bytesRead = fileChannel.read(byteBuffer);  

22.   }  

23. randomAccessFile.close(); 
```



FileChannel的常用方法

1. **read方法**

   从Channel中读取数据，用Buffer存储数据。

2.  **write方法**

   从Buffer向Channel中写入数据，通常在while中写入，因为无法保证write()方法一次能向FileChannel写入多少字节，因此需要重复调用write()方法，直到Buffer中已经没有尚未写入通道的字节。

3. **close方法**

   关闭通道。

4. **position方法**

   获取FileChannel的当前位置。也可以用position(long pos)来设置位置。

注意：

- 读的时候将position设置为ByteBuffer的capablity外会返回-1（文件结束标志）。

- 写的时候将position设置为ByteBuffer的capablity外不会报错，会从指定位置写，但是会将文件撑大，可能导致“文件空洞”，磁盘上物理文件中写入的数据间有空隙。

5. **size方法**

   返回该实例所关联文件的大小。

6. **truncate方法**

   截取一个文件。截取文件时，文件将中指定长度后面的部分将被删除。

7. **force方法**

   将通道里尚未写入磁盘的数据强制写到磁盘上。出于性能方面的考虑，操作系统会将数据缓存在内存中，所以无法保证写入到FileChannel里的数据一定会即时写到磁盘上。要保证这一点，需要调用force()方法。

   force()方法有一个boolean类型的参数，指明是否同时将文件元数据（权限信息等）写到磁盘上。



### 2. DatagramChannel

能通过UDP读写网络中的数据。因为UDP是无连接的网络协议，所以不能像其它通道那样循环读取和写入。它发送和接收的是数据包。

```
1.  DatagramChannel channel = DatagramChannel.open();  

2.  channel.socket().bind(new InetSocketAddress(9999));  

3.  //发送数据  

4.  String newData = "New String to write to file..." + System.currentTimeMillis();  

5.  ByteBuffer buf = ByteBuffer.allocate(48);  

6.  buf.clear();  

7.  buf.put(newData.getBytes());  

8.  buf.flip();  

9.  int bytesSent = channel.send(buf, new InetSocketAddress("jenkov.com", 80));  

10. //接收数据  

11. ByteBuffer buf = ByteBuffer.allocate(48);  

12. buf.clear();  

13. channel.receive(buf); 
```

UDP在数据传送方面没有任何保证，不会通知你发出的数据包是否已收到。

由于UDP是无连接的，连接到特定地址并不会像TCP通道那样创建一个真正的连接。而是锁住DatagramChannel，让其只能从特定地址收发数据。



### 3. SocketChannel（TCP客户端）

能通过UDP读写网络中的数据。

创建SocketChannel的两种方式

- 打开一个SocketChannel并连接到互联网上的某台服务器。
- 一个新连接到达ServerSocketChannel时，会创建一个SocketChannel。



阻塞型的Channel（默认阻塞）

```
1.  //连接通道  

2.  SocketChannel socketChannel = SocketChannel.open();  

3.  socketChannel.connect(new InetSocketAddress("http://jenkov.com", 80));  

4.  //写入数据  

5.  String newData = "New String to write to file..." + System.currentTimeMillis();  

6.  ByteBuffer buf = ByteBuffer.allocate(48);  

7.  buf.clear();  

8.  buf.put(newData.getBytes());  

9.  buf.flip();  

10. while(buf.hasRemaining()) {  

11.     channel.write(buf);  

12. }  

13. //读取数据  

14. ByteBuffer buf = ByteBuffer.allocate(48);  

15. int bytesRead = socketChannel.read(buf);  

16. //关闭通道  

17. socketChannel.close(); 
```



设置为非阻塞模式

```
1.  socketChannel.configureBlocking(false);  

2.  socketChannel.connect(new InetSocketAddress("http://jenkov.com", 80));  

3.  //connect()方法可能在连接建立之前就返回了  

4.  while(! socketChannel.finishConnect() ){  

5.      //wait, or do something else...  

6.  }  
```



在非阻塞模式下

- write()方法在尚未写出任何内容时可能就返回了。所以需要在循环中调用write()。前面已经有例子了，这里就不赘述了。
- read()方法在尚未读取到任何数据时可能就返回了。所以需要关注它的int返回值，它会告诉你读取了多少字节。



非阻塞模式与选择器搭配会工作的更好，通过将一或多个SocketChannel注册到Selector，可以询问选择器哪个通道已经准备好了读取，写入等。



### 4. ServerSocketChannel（TCP服务端）

可以监听新进来的TCP连接，像Web服务器那样。对每一个新进来的连接都会创建一个SocketChannel。

阻塞模式：

```
1.  ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();  

2.  serverSocketChannel.socket().bind(new InetSocketAddress(9999));  

3.  while(true){  

4.      SocketChannel socketChannel = serverSocketChannel.accept();  

5.      //do something with socketChannel...  

6.  }  
```



非阻塞模式：

```
1.  ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();  

2.  serverSocketChannel.socket().bind(new InetSocketAddress(9999));  

3.  serverSocketChannel.configureBlocking(false);  

4.  while(true){  

5.      SocketChannel socketChannel = serverSocketChannel.accept();  

6.      //在非阻塞模式下，accept() 方法会立刻返回，如果还没有新进来的连接,返回的将是null  

7.      if(socketChannel != null){  

8.          //do something with socketChannel...  

9.      }  

10. }  
```



通道之间的传输：

如果两个通道有一个是FileChannel就可以通过transferFrom()和transferTo()方法进行数据传输。

两个方法作用是一样的，只是调用对象不一样



示例代码如下：

```
1.  RandomAccessFile fromFile = new RandomAccessFile("fromFile.txt", "rw");  

2.  FileChannel      fromChannel = fromFile.getChannel();  

3.  RandomAccessFile toFile = new RandomAccessFile("toFile.txt", "rw");  

4.  FileChannel      toChannel = toFile.getChannel();  

5.  long position = 0;  

6.  long count = fromChannel.size();  

7.  toChannel.transferFrom(position, count, fromChannel); 
```

如果源通道的剩余空间小于 count 个字节，则所传输的字节数要小于请求的字节数。

此外要注意，在SoketChannel的实现中，SocketChannel只会传输此刻准备好的数据（可能不足count字节）。因此，SocketChannel可能不会将请求的所有数据(count个字节)全部传输到FileChannel中。