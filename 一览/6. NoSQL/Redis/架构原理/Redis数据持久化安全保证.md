**怎么保证生成rdb文件和aof文件的安全性？**



很多时候我们需要持久化数据也就是将内存中的数据写入到硬盘里面，大部分原因是为了之后重用数据（比如重启机器、机器故障之后恢复数据），或者是为了防止系统故障而将数据备份到一个远程位置。



先简单介绍下redis持久化会和磁盘、内存有哪些交互过程？怎么保证数据安全？

![img](http://pcc.huitogo.club/7e5323327b52807cd2e115c69b09147a)



介绍下就是：

1. 客户端向数据库 发送写命令 (数据在客户端的内存中)
2. 数据库 接收 到客户端的 写请求 (数据在服务器的内存中)
3. 数据库 调用系统 API 将数据写入磁盘 (数据在内核缓冲区中)
4. 操作系统将 写缓冲区 传输到 磁盘控控制器 (数据在磁盘缓存中)
5. 操作系统的磁盘控制器将数据 写入实际的物理媒介中 (数据在磁盘中)



注意: 上面的过程其实是 极度精简 的，在实际的操作系统中，缓存和缓冲区会比这多得多…



如果我们故障仅仅涉及到 软件层面 (该进程被管理员终止或程序崩溃) 并且没有接触到内核，那么在 上述步骤 3 成功返回之后，我们就认为成功了。即使进程崩溃，操作系统仍然会帮助我们把数据正确地写入磁盘。

如果我们考虑 停电/ 火灾 等 更具灾难性 的事情，那么只有在完成了第 5 步之后，才是安全的。



所以我们可以总结得出数据安全最重要的阶段是：步骤三、四、五，即：

1. 数据库软件调用写操作将用户空间的缓冲区转移到内核缓冲区的频率是多少？
2. 内核多久从缓冲区取数据刷新到磁盘控制器？
3. 磁盘控制器多久把数据写入物理媒介一次？



注意： 如果真的发生灾难性的事件，我们可以从上图的过程中看到，任何一步都可能被意外打断丢失，所以只能 尽可能地保证数据的安全，这对于所有数据库来说都是一样的。



我们从 第三步 开始。Linux 系统提供了清晰、易用的用于操作文件的 POSIX file API，20 多年过去，仍然还有很多人对于这一套 API 的设计津津乐道，我想其中一个原因就是因为你光从 API 的命名就能够很清晰地知道这一套 API 的用途：

```
1. int open(const char *path, int oflag, .../*,mode_t mode */); 

2. int close (int filedes);int remove( const char *fname ); 

3. ssize_t write(int fildes, const void *buf, size_t nbyte); 

4. ssize_t read(int fildes, void *buf, size_t nbyte); 
```



所以，我们有很好的可用的 API 来完成 第三步，但是对于成功返回之前，我们对系统调用花费的时间没有太多的控制权。



然后我们来说说 **第四步**。我们知道，除了早期对电脑特别了解那帮人 (操作系统就这帮人搞的)，实际的物理硬件都不是我们能够 **直接操作** 的，都是通过 **操作系统调用** 来达到目的的。为了防止过慢的 I/O 操作拖慢整个系统的运行，操作系统层面做了很多的努力，譬如说 **上述第四步** 提到的 **写缓冲区**，并不是所有的写操作都会被立即写入磁盘，而是要先经过一个缓冲区，默认情况下，Linux 将在 30 秒 后实际提交写入。



但是很明显，30 秒 并不是 Redis 能够承受的，这意味着，如果发生故障，那么最近 30 秒内写入的所有数据都可能会丢失。幸好 PROSIX API 提供了另一个解决方案：fsync，该命令会**强制**内核将 缓冲区**写入**磁盘，但这是一个非常消耗性能的操作，每次调用都会 阻塞等待 直到设备报告 IO 完成，所以一般在生产环境的服务器中，Redis 通常是每隔 1s 左右执行一次 fsync 操作。



到目前为止，我们了解到了如何控制 第三步 和 第四步，但是对于 第五步，我们 完全无法控制。也许一些内核实现将试图告诉驱动实际提交物理介质上的数据，或者控制器可能会为了提高速度而重新排序写操作，不会尽快将数据真正写到磁盘上，而是会等待几个多毫秒。这完全是我们无法控制的。