官方FAQ表示，因为Redis是基于内存的操作，**CPU不是Redis的瓶颈，Redis的瓶颈最有可能是机器内存的大小或者网络带宽**。既然单线程容易实现，而且CPU不会成为瓶颈，那就顺理成章地采用单线程的方案了（毕竟采用多线程会有很多麻烦！）



需要注意的是，redis采用的是使用**单线程高效的处理多个连接请求**，一个正式的RedisServer运行的时候肯定是不止一个线程的，例如Redis进行持久化的时候会以子进程或者子线程的方式执行。



虽然我们我们使用单线程的方式是无法发挥多核CPU性能，不过我们可以通过在单机开多个Redis 实例来完善。



但是在**Redis6.0后开始引入了多线程**

Redis6.0 引入多线程主要是为了提高网络 IO 读写性能，因为这个算是 Redis 中的一个性能瓶颈（Redis 的瓶颈主要受限于内存和网络）。



虽然，Redis6.0 引入了多线程，但是 Redis 的多线程只是在网络数据的读写这类耗时操作上使用了， 执行命令仍然是单线程顺序执行。因此，你也不需要担心线程安全问题。



Redis6.0 的多线程默认是禁用的，只使用主线程。如需开启需要修改 redis 配置文件 redis.conf ：

```
// 开启多线程
1. io-threads-do-reads yes 
```



开启多线程后，还需要设置线程数，否则是不生效的。同样需要修改 redis 配置文件 redis.conf :

```
 #官网建议4核的机器建议设置为2或3个线程，8核的建议设置为6个线程 
1. io-threads 4 
```