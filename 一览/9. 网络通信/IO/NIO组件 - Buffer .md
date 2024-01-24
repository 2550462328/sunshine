数据读写步骤整体分为以下四部：

1. 写入数据到Buffer

2. 调用flip()方法

3. 从Buffer中读取数据

4. 调用clear()方法或者compact()方法



当向buffer写入数据时，buffer会记录下写了多少数据。**一旦要读取数据，需要通过flip()方法将Buffer从写模式切换到读模式**。在读模式下，可以读取之前写入到buffer的所有数据。

一旦读完了所有的数据，就需要清空缓冲区，让它可以再次被写入。有两种方式能清空缓冲区：调用clear()或compact()方法。clear()方法会清空整个缓冲区。compact()方法只会清除已经读过的数据。任何未读的数据都被移到缓冲区的起始处，新写入的数据将放到缓冲区未读数据的后面。



### 1. Buffers的三个属性

- capacity

- position

- limit



position和limit的含义取决于Buffer处在读模式还是写模式。不管Buffer处在什么模式，capacity的含义总是一样的。

区别在于你写的数据不一定能占满整个ByteBuffer，你写的时候上限是ByteBuffer的上限，没有问题，但是你读的时候，上限肯定是你写入的长度，也就是写操作结束时的position。

![img](http://pcc.huitogo.club/5e12fd2f644a51324c7668a74393d97f)



#### 1.1 capacity

作为一个内存块，Buffer有一个固定的大小值，也叫“capacity”。你只能往里写capacity个byte、long，char等类型。一旦Buffer满了，需要将其清空（通过读数据或者清除数据）才能继续写数据往里写数据。



#### 1.2 position

当你写数据到Buffer中时，position表示当前的位置。初始的position值为0。当一个byte、long等数据写到Buffer后，position会向前移动到下一个可插入数据的Buffer单元。position最大可为capacity – 1。

当读取数据时，也是从某个特定位置读。当将Buffer从写模式切换到读模式，position会被重置为0。当从Buffer的position处读取数据时，position向前移动到下一个可读的位置。



#### 1.3 limit

在写模式下，Buffer的limit表示你最多能往Buffer里写多少数据。写模式下，limit等于Buffer的capacity。

当切换Buffer到读模式时，limit表示你最多能读到多少数据。因此，当切换Buffer到读模式时，limit会被设置成写模式下的position值。换句话说，你能读到之前写入的所有数据（limit被设置成已写数据的数量，这个值在写模式下就是position）



### 2. Buffer的类型

- ByteBuffer
- MappedByteBuffer
- CharBuffer
- DoubleBuffer
- FloatBuffer
- IntBuffer
- LongBuffer
- ShortBuffer

除了MappedByteBuffer，其他区别都是Buffer里面数据类型不同，这些Buffer可以相互转换。



### 3. ByteBuffer的实例化

ByteBuffer类提供了4个静态工厂方法来获得ByteBuffer的实例：

1. allocate(int capacity)

   从堆空间中分配一个容量大小为capacity的byte数组作为缓冲区的byte数据存储器

2. allocateDirect(int capacity)

   是不使用JVM堆栈而是通过操作系统来创建内存块用作缓冲区，它与当前操作系统能够更好的耦合，因此能进一步提高I/O操作速度。但是分配直接缓冲区的系统开销很大，因此只有在缓冲区较大并长期存在，或者需要经常重用时，才使用这种缓冲区

3. wrap(byte[] array)

   这个缓冲区的数据会存放在byte数组中，bytes数组或buff缓冲区任何一方中数据的改动都会影响另一方。其实ByteBuffer底层本来就有一个bytes数组负责来保存buffer缓冲区中的数据，通过allocate方法系统会帮你构造一个byte数组

4. wrap(byte[] array, int offset, int length)

   在上一个方法的基础上可以指定偏移量和长度，这个offset也就是包装后byteBuffer的position，而length呢就是limit-position的大小，从而我们可以得到limit的位置为length+position(offset)



### 4. Buffer的主要方法

#### 4.1 分配空间

这是分配一个可存储1024个字符的CharBuffer：

```
CharBuffer buf = CharBuffer.allocate(1024);
```



#### 4.2 写数据

从Channel写到Buffer。

```
int bytesRead = inChannel.read(buf); //read into buffer.
```



通过Buffer的put()方法写到Buffer里。

```
buf.put(127);
```

put方法有很多版本，允许你以不同的方式把数据写入到Buffer中。例如，写到一个指定的位置，或者把一个字节数组写入到Buffer。



#### 4.3 读取数据

从Buffer读取数据到Channel。

```
int bytesWritten = inChannel.write(buf);
```

​	使用get()方法从Buffer中读取数据。

```
byte aByte = buf.get();
```

​	get方法有很多版本，允许你以不同的方式从Buffer中读取数据。例如，从指定		position读取，或者从Buffer中读取数据到字节数组。



#### 4.4 设置position

1）flip方法

flip方法将Buffer从写模式切换到读模式。调用flip()方法会将position设回0，并将limit设置成之前position的值。

换句话说，position现在用于标记读的位置，limit表示之前写进了多少个byte、char等 现在能读取多少个byte、char等。



2）rewind方法

Buffer.rewind()将position设回0，所以你可以重读Buffer中的所有数据。limit保		持不变，仍然表示能从Buffer中读取多少个元素（byte、char等）。



3） clear()与compact()方法

一旦读完Buffer中的数据，需要让Buffer准备好再次被写入。可以通过clear()或			compact()方法来完成。

如果调用的是clear()方法，position将被设回0，limit被设置成capacity的值。换句		话说，Buffer被清空了。Buffer中的数据并未清除，只是这些标记告诉我们可以从		哪里开始往Buffer里写数据。

如果Buffer中有一些未读的数据，调用clear()方法，数据将“被遗忘”，意味着不再		有任何标记会告诉你哪些数据被读过，哪些还没有。

如果Buffer中仍有未读的数据，且后续还需要这些数据，但是此时想要先先写些数		据，那么使用compact()方法。

compact()方法将所有未读的数据拷贝到Buffer起始处。然后将position设到最后		一个未读元素正后面。limit属性依然像clear()方法一样，设置成capacity。现在			Buffer准备好写数据了，但是不会覆盖未读的数据。



4）mark()与reset()方法

通过调用Buffer.mark()方法，可以标记Buffer中的一个特定position。之后可以通		过调用Buffer.reset()方法恢复到这个position。



#### 4.5 比较方法

使用equals()和compareTo()方法两个Buffer。

1）equals()

当满足下列条件时，表示两个Buffer相等：

```
有相同的类型（byte、char、int等）。

Buffer中剩余的byte、char等的个数相等。

Buffer中所有剩余的byte、char等都相同。
```

​		如你所见，equals只是比较Buffer的一部分，不是每一个在它里面的元素都比较。		实际上，它只比较Buffer中的剩余元素。



2）compareTo()方法

compareTo()方法比较两个Buffer的剩余元素(byte、char等)，如果满足下列条		件，则认为一个Buffer“小于”另一个Buffer：

```
第一个不相等的元素小于另一个Buffer中对应的元素 。

所有元素都相等，但第一个Buffer比另一个先耗尽(第一个Buffer的元素个数比另一个少)。
```



### 5. Buffers和Channel的分散和聚合

分散和聚合都是按数组中ByteBuffer的顺序执行的。

#### 5.1 分散（Scattering Reads）

Scattering Reads是指数据从一个channel读取到多个buffer中。

```
1.  ByteBuffer header = ByteBuffer.allocate(128);  

2.  ByteBuffer body   = ByteBuffer.allocate(1024);  

3.  ByteBuffer[] bufferArray = { header, body };  

4.  channel.read(bufferArray);  
```

Scattering Reads在移动下一个buffer前，必须填满当前的buffer，这也意味着它		不适用于动态消息(译者注：消息大小不固定)。换句话说，如果存在消息头和消息		体，消息头必须完成填充（例如128byte），Scattering Reads才能正常工作。



#### 4.2 聚合（Gathering Writes）

Gathering Writes是指数据从多个buffer写入到同一个channel。

```
1.  ByteBuffer header = ByteBuffer.allocate(128);  

2.  ByteBuffer body   = ByteBuffer.allocate(1024);  

3.  //write data into buffers  

4.  ByteBuffer[] bufferArray = { header, body };  

5.  channel.write(bufferArray); 
```

有position和limit之间的数据才会被写入，如果一个buffer的容量为128byte，但是仅仅包含58byte的数据，那么这58byte的数据将被写入到channel中。因此与Scattering Reads相反，Gathering Writes能较好的处理动态消息。



使用Channel和Buffer操作的实例：

```
1.  public class BufferToText {    

2.      private static final int BSIZE=1024;    

3.      public static void main(String[] args) throws Exception {    

4.          //写入file中    

5.          FileChannel fc = new FileOutputStream("date2.txt").getChannel();    

6.          //将字符转换为字节数组，在放入字节缓冲器中。最后写入到通道中    

7.          fc.write(ByteBuffer.wrap("my love我的爱".getBytes()));         

8.          fc.close();    

9.          fc=new FileInputStream("date2.txt").getChannel();    

10.         ByteBuffer buffer=ByteBuffer.allocate(BSIZE);//定义一个指定大小的字节缓冲区    

11.         fc.read(buffer);    

12.         buffer.flip();    

13.         //打印读取到的字节    

14.         //System.out.print(buffer.asCharBuffer());//这种方法没什么用，会出现乱码。    

15.         //我们需要系统的指定默认的字符集    

16.         buffer.rewind();    

17.         String encoding =System.getProperty("file.encoding");//获取文件的字节码    

18.         System.out.println("Decoding using:"+encoding+":"+Charset.forName(encoding).decode(buffer));        

19.         //或者我们也可以使用编码来向文件中写入字符    

20.         fc=new FileOutputStream("date3.txt").getChannel();    

21.         fc.write(ByteBuffer.wrap("我是一份大菠菜。".getBytes("UTF-8")));    

22.         fc.close();    

23.         //接下来我们进行读文件    

24.         fc=new FileInputStream("date3.txt").getChannel();    

25.         buffer.clear();    

26.         fc.read(buffer);    

27.         buffer.flip();    

28.         System.out.print(buffer.asCharBuffer());    

29.     }    

30. }  
```



### 6. Buffer的选择

有两种类型，**DirectByteBuffer**和**HeapByteBuffer**



![img](http://pcc.huitogo.club/2dc75a02e782b74c6c90a6ec58c46b60)



ByteBuffer.allocate返回的是HeapByteBuffer

```
1.  public static ByteBuffer allocate(int capacity) {  

2.      if (capacity < 0)  

3.          throw new IllegalArgumentException();  

4.      return new HeapByteBuffer(capacity, capacity);  

5.  }  
```



ByteBuffer.allocateDirect返回的是DirectByteBuffer

```
1.  public static ByteBuffer allocateDirect(int capacity) {  

2.      return new DirectByteBuffer(capacity);  

3.  }  
```



通常情况下，操作系统的一次写操作分为两步：

1. 将数据从用户空间拷贝到系统空间。

2. 从系统空间往网卡写。




同理，读操作也分为两步：

1. 将数据从网卡拷贝到系统空间；

2. 将数据从系统空间拷贝到用户空间。




使用了DirectByteBuffer，一般来说可以减少一次系统空间到用户空间的拷贝。但Buffer创建和销毁的成本更高，更不宜维护，通常会用内存池来提高性能。

如果数据量比较小的中小应用情况下，可以考虑使用heapBuffer；反之可以用directBuffer。