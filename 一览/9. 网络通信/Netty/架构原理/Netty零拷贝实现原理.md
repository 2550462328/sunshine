零拷贝是指避免在用户态（User-space）与内核态（Kernel-space）之间来回拷贝数据的技术。



#### 1. 传统IO

传统 IO（InputStream/OutputStream）读取数据并通过网络发送数据的流程，如下图：

![img](http://pcc.huitogo.club/af17f4fa9ee7206bce08784ecb12abfd)



1. 当用户空间使用 read() 询问内核空间时，会进行一次上下文的切换（从用户态切换到内核态）。然后内核空间通过 sys_read() 或等价的方法（与操作系统相关）从磁盘中读取数据（ask for data）。使用 DMA(Direct Memory Access，直接内存存取)引擎执行第一次拷贝：从磁盘空间拷贝数据到内核缓冲区中。
2. 请求的数据从内核读缓冲区拷贝到用户缓冲区（cope data to user buffer），这里又进行了一次上下文切换（从内核态切换到用户态）现在待读取的数据已存储在用户空间内存的缓冲区。至此，完成了一次 IO 的读取过程。
3. wirte() 调用将用户空间数据拷贝到内核缓冲区（与上面的不同，这次指的是 socket 缓冲区） 中，也会进行一次上下文切换（从用户态切换到内核态）。然后再由内核缓冲区将数据 writes data 到目标文件。
4. 最终 write returns（调用返回）会进行第四次上下文切换（从内核态切换到用户态）。



磁盘到内核空间属于 DMA 拷贝，用户空间与内核空间之间的数据传输并没有类似 DMA 这种可以不需要 CPU 参与的传输方式，因此用户空间与内核空间之间的数据传输是需要 CPU 全程参于的。

DMA(Direct Memory Access，直接内存存取)，原理是外部设备不需要通过 CPU 就可以直接与系统内存交换数据；所以也就有了零拷贝技术，避免不必要的 CPU 数据拷贝过程。



#### 2. NIO 零拷贝（方案一）

NIO 的零拷贝由 transferTo 方法实现。transferTo 方法将数据从 FileChannel 对象传送到可写的字节通道（如Socket Channel等）。在 transferTo 方法内部实现中，由 native 方法 transferTo0 来实现，它依赖操作系统底层的支持。在 UNIX 和 Linux 系统中，调用这个方法会引起 sendfile() 系统调用，实现了数据直接从内核的读缓冲区传输到套接字缓冲区，避免了用户态（User-space）与内核态（Kernel-space）之间的数据拷贝。

![img](http://pcc.huitogo.club/2821fa36ebe7cead366d5b3d6bcd314a)



客户端发起 sendfile() 后，进行上下文切换到内核状态。内核向磁盘发起 ask for data，使用的是操作系统的 transferTo 方法调用触发 DMA 引擎将文件上下文信息拷贝到内核缓冲区中。与上面的不同是，此时，不会再将数据发送给用户缓冲区（减少复制和上下文切换）。而是 write data to target socket buffer（将数据从内核缓冲区拷贝到目标 socket（套接字） 相关联的缓冲区）

相比于传统 IO ，使用 NIO 零拷贝后。我们将上下文切换次数从4次减少到了2次。将数据拷贝次数从4次减少到3次（其中一次涉及了 CPU，另外2次是 DMA 直接存取）。

```
/**
 * filechannel进行文件复制（零拷贝）
 *
 * @param fromFile 源文件
 * @param toFile   目标文件
 */
public static void fileCopyWithFileChannel(File fromFile, File toFile) {
    try (// 得到fileInputStream的文件通道
         FileChannel fileChannelInput = new FileInputStream(fromFile).getChannel();
         // 得到fileOutputStream的文件通道
         FileChannel fileChannelOutput = new FileOutputStream(toFile).getChannel()) {
 
        //将fileChannelInput通道的数据，写入到fileChannelOutput通道
        fileChannelInput.transferTo(0, fileChannelInput.size(), fileChannelOutput);
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```



#### 3. NIO 零拷贝（方案二）

如果操作系统底层支持 gather 操作，可以进一步减少内核中的数据拷贝，在 Linux2.4 以及更高版本的内核中，socket 缓冲区描述符已被修改用来适应这个需求。这种方式不但减少了上下文切换，同时消除了 CPU 参于的重复的数据拷贝。

![img](http://pcc.huitogo.club/d02fd85c856fb7fc61f998adad98eec4)



用户使用方式不变，依旧通过 TransferTo 方法，但是方法的内部实现发生了变化：

1. transferTo 方法调用触发 DMA 引擎将文件上下文信息拷贝到内核缓冲区。
2. 数据不会拷贝到套接字缓冲区，只是数据的描述符（包括数据位置和长度）被拷贝到套接字缓冲区。DMA 引擎直接将数据从内核缓冲区拷贝到 Hardware，这样就减少了最后一次需要消耗 CPU 的拷贝操作。



#### 4. NIO 零拷贝适用场景

1. 文件较大，读写较慢，追求速度；
2. JVM 内存不足，不能加载太大数据；
3. 内存带宽不够，即存在其它程序或线程存在大量的 IO 操作，导致带宽减小。



#### 5. 直接内存映射

在不需要进行数据文件操作时，可以使用NIO的零拷贝。但如果既需要IO速度，又需要进行数据操作，则需要使用NIO的直接内存映射。

Linux 提供的 mmap 系统调用, 它可以将一段用户空间内存映射到内核空间, 当映射成功后, 用户对这段内存区域的修改可以直接反映到内核空间；同样地， 内核空间对这段区域的修改也直接反映用户空间。正因为有这样的映射关系, 就不需要在用户态(User-space) 与内核态 (Kernel-space) 之间拷贝数据， 提高了数据传输的效率，这就是内存直接映射技术。



##### 5.1 NIO的直接内存映射

JDK1.4 加入了 NIO 机制和直接内存，目的是防止 Java堆和 Native堆之间数据复制带来的性能损耗，此后 NIO可以使用 Native的方式直接在 Native堆分配内存。

![img](http://pcc.huitogo.club/4caf028decacd5bcd641f5f933b542df)

```
//背景：堆内数据在flush到远程时，会先复制到Native 堆，然后再发送；直接移到堆外就更快了。
static int write(FileDescriptor paramFileDescriptor,ByteBuffer paramByteBuffer, long paramLong, NativeDispatcher paramNativeDispatcher) throws IOException{
    //如果是直接缓冲区，则不用复制，直接写出
    if((paramByteBuffer instanceof DirectBuffer)){
        Return writeFromNativeBuffer(paramFileDescriptor, paramByteBuffer, paramLong, paramNativeDispatcher);
    }
    //如果不是直接缓冲区，会对缓冲数据进行赋值
    int i = paramByteBuffer.position();
    int j = paramByteBuffer.limit();
    assert (i <= j);
    int k = i <= j ? j-i : 0;
    //赋值的数据放入 localByteBuffer 
    ByteBuffer localByteBuffer = Util.getTemporaryDirectBuffer(k);
    try{
        localByteBuffer.put(paramByteBuffer);
        localByteBuffer.flip();
        paramByteBuffer.position(i);
        int m = writeFromNativeBuffer(paramFileDescriptor,localByteBuffer,paramLong,paramNativeDispatcher);
        if(m > 0){
            paramByteBuffer.position(i + m);
        }
        return m;
    } finally {
        Util.offerFirstTemporaryDirectBuffer(localByteBuffer);
    }
}
```



##### 5.2 直接内存的创建

在 ByteBuffer 有两个子类，HeapByteBuffer 和 DirectByteBuffer。前者是存在于 JVM堆中的，后者是存在于 Native堆中的。

![img](http://pcc.huitogo.club/ec5c1916a1d899c13cee46816116cf71)

```
//申请堆内存
public static ByteBuffer allocate(int capacity) {
    if (capacity < 0)
        throw new IllegalArgumentException();
    return new HeapByteBuffer(capacity, capacity);
}

//申请直接内存
public static ByteBuffer allocateDirect(int capacity) {
    return new DirectByteBuffer(capacity);
}
```



#### 6. 使用直接内存的原因

1. 对垃圾回收停顿的改善。因为full gc时，垃圾收集器会对所有分配的堆内内存进行扫描，垃圾收集对 Java应用造成的影响，跟堆的大小是成正比的。过大的堆会影响 Java应用的性能。如果使用堆外内存的话，堆外内存是直接受操作系统管理。这样做的结果就是能保持一个较小的 JVM堆内存，以减少垃圾收集对应用的影响。（full gc时会触发堆外空闲内存的回收）
2. 减少了数据从 JVM拷贝到 native堆的次数，在某些场景下可以提升程序 I/O的性能。
3. 以突破 JVM内存限制，操作更多的物理内存。当直接内存不足时会触发full gc，排查full gc的时候，一定要考虑。



#### 7. 使用直接内存的问题

1. 堆外内存难以控制，如果内存泄漏，那么很难排查（VisualVM 可以通过安装插件来监控堆外内存）。
2. 堆外内存只能通过序列化和反序列化来存储，保存对象速度比堆内存慢，不适合存储很复杂的对象。一般简单的对象或者扁平化的比较适合。
3. 直接内存的访问速度（读写方面）会快于堆内存。在申请内存空间时，堆内存速度高于直接内存。



直接内存适合申请次数少，访问频繁的场合。如果内存空间需要频繁申请，则不适合直接内存。



#### 8. NIO 的直接内存映射

NIO 中一个重要的类：MappedByteBuffer nio引入的文件内存映射方案，读写性能极高。MappedByteBuffer 将文件直接映射到内存。可以映射整个文件，如果文件比较大的话可以考虑分段进行映射，只要指定文件的感兴趣部分就可以。

由于 MappedByteBuffer 申请的是直接内存，因此不受 Minor GC 控制，只能在发生 Full GC 时才能被回收，因此 Java提供了DirectByteBuffer 类来改善这一情况。它是 MappedByteBuffer 类的子类，同时它实现了 DirectBuffer 接口，维护一个 Cleaner对象来完成内存回收。因此它既可以通过 Full GC来回收内存，也可以调用 clean()方法来进行回收。



直接内存映射代码演示 ：

```
static final int BUFFER_SIZE = 1024;
 
/**
 * 使用直接内存映射读取文件
 * @param file
 */
public static void fileReadWithMmap(File file) {
 
    long begin = System.currentTimeMillis();
    byte[] b = new byte[BUFFER_SIZE];
    int len = (int) file.length();
    MappedByteBuffer buff;
    try (FileChannel channel = new FileInputStream(file).getChannel()) {
        // 将文件所有字节映射到内存中。返回MappedByteBuffer
        buff = channel.map(FileChannel.MapMode.READ_ONLY, 0, channel.size());
        for (int offset = 0; offset < len; offset += BUFFER_SIZE) {
            if (len - offset > BUFFER_SIZE) {
                buff.get(b);
            } else {
                buff.get(new byte[len - offset]);
            }
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
    long end = System.currentTimeMillis();
    System.out.println("time is:" + (end - begin));
}
 
/**
 * HeapByteBuffer读取文件
 * @param file
 */
public static void fileReadWithByteBuffer(File file) {
 
    long begin = System.currentTimeMillis();
    try(FileChannel channel = new FileInputStream(file).getChannel();) {
        // 申请HeapByteBuffer
        ByteBuffer buff = ByteBuffer.allocate(BUFFER_SIZE);
        while (channel.read(buff) != -1) {
            buff.flip();
            buff.clear();
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
    long end = System.currentTimeMillis();
    System.out.println("time is:" + (end - begin));
}
```