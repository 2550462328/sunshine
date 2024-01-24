### 1. 内存优化

1. **关掉对web.xml的监视，把jsp提前编辑成Servlet。**

   

2. **jvm优化**

有富余物理内存的情况，加大tomcat使用的jvm的内存。

服务器所能提供CPU、内存、硬盘的性能对处理能力有决定性影响：

1. 对于高并发情况下会有大量的运算，那么CPU的速度会直接影响到处理速度。
2. b内存在大量数据处理的情况下，将会有较大的内存容量需求，可以用-Xmx -Xms - XX:MaxPermSize等参数对内存不同功能块进行划分。我们之前就遇到过内存分配不足，导致 虚拟机一直处于full GC，从而导致处理能力严重下降。
3. c硬盘主要问题就是读写性能，当大量文件进行读写时，磁盘极容易成为性能瓶颈。最好的办法 还是利用下面提到的缓存。



Tomcat的默认内存配置比较低,不用说大项目,就算是小项目,并发量达到一定程度也就可能会抛出OutOfMemoryError异常,为了解决这个问题,我们要修改JVM的一些配置,在tomcat的bin目录下的catalina配置文件中,配置Xms和Xmx,也就是Java虚拟机初始化时堆的最小内存和最大内存,这俩值通常会配置成一样,这样GC不必再为扩展内存空间而消耗性能.



除了这两个,还可以配置XX:PermSize和XX:MaxPermSize,它们是Java虚拟机永久代大小和最大值,除了这几个参数

还可以再根据具体需要配置其他参数。

```
setJAVA_OPTS=-server -Xms1024m -Xmx2048m -Xss512K -XX:PermSize=128m-XX:MaxPermSize=256m  
```



3. **利用缓存和压缩**

对于静态页面最好是能够缓存起来，这样就不必每次从磁盘上读。这里我们采用了Nginx作为 缓存服务器，将图片、css、js文件都进行了缓存，有效的减少了后端tomcat的访问。

另外，为了能加快网络传输速度，开启gzip压缩也是必不可少的。但考虑到tomcat已经需要处 理很多东西了，所以把这个压缩的工作就交给前端的Nginx来完成。

除了文本可以用gzip压缩，其实很多图片也可以用图像处理工具预先进行压缩，找到一个平衡 点可以让画质损失很小而文件可以减小很多。曾经我就见过一个图片从300多kb压缩到几十 kb，自己几乎看不出来区别。



### 2. 配置优化

配置优化,主要有三方面:

#### 2.1 Connector 优化

Connector是连接器，它负责接收客户的请求，以及向客户端回送响应的消息。

默认情况下Tomcat只支持200线程访问，超过这个数量的连接将被等待甚至超时放弃，所以我们需要提高这方面的处理能力。

修改这部分配置需要修改conf下的server.xml，找到Connector标签项,修改protocol,默认的协议类型是BIO,也就是阻塞式I/O操作，简单项目及应用可以采用BIO。

第二种协议类型是NIO,它就是一种基于缓冲区是、并能提供非阻塞I/O操作的java API,它有更好的并发运行性能. NIO更适合后台需要耗时完成请求的操作

第三种协议类型是APR,它主要可以提高Tomcat对静态文件的处理性能.



选择哪个协议也是根据实际项目进行配置.

1. Bio（default）：即阻塞式I/O操作，表示Tomcat使用的是传统的Java I/O操作(即java.io包及其子包)。Tomcat在默认情况下，就是以bio模式运行的。遗憾的是，就一般而言，bio模式是三种运行模式中性能最低的一种。
2. NIO：是Java SE 1.4及后续版本提供的一种新的I/O操作方式(即java.nio包及其子包)。Java nio是一个**基于缓冲区**、并能提供**非阻塞**I/O操作的Java API，因此nio也被看成是non-blocking I/O的缩写。它拥有比传统I/O操作(bio)更好的**并发运行性能**。
3. APR（Apache Portable Runtime/Apache可移植运行时）：是Apache HTTP服务器的支持库。你可以简单地理解为，Tomcat将以JNI的形式调用Apache HTTP服务器的核心动态链接库来处理文件读取或网络传输操作，从而大大地**提高Tomcat对静态文件的处理性能**。Tomcat apr也是在**Tomcat上运行高并发应用的首选模式**。

4. 自定义：需要继承org.apache.coyote.ProtocolHandler接口。（TODO）



除了这个协议类型,还有一个非常重要的参数要改,就是maxThreads,就是当前连接器能够处理同时请求的最大数目.这个数目也并非越大越好,它也受操作系统等硬件制约,所以这个值要根据压力测试后实际数据进行配置。

```
<Connector port="8080" protocol="HTTP/1.1"connectionTimeout="20000"
redirectPort="8443"acceptCount="500" maxThreads="400" />
```

其中：

• maxThreads：tomcat可用于请求处理的最大线程数，默认是200

• minSpareThreads：tomcat初始线程数，即最小空闲线程数

• maxSpareThreads：tomcat最大空闲线程数，超过的会被关闭

• acceptCount：当所有可以使用的处理请求的线程数都被使用时，可以放到处理队列中的请求数，超过这个数的请求将不予处理.默认100



#### 2.2 线程池

使用线程池的好处在于减少了创建销毁线程的相关消耗，而且可以提高线程的使用效率。使用线程池就在Service标签中配置Executor就可以了

```
<Executor name="tomcatThreadPool" namePrefix="req-exec-"maxThreads="1000"
 minSpareThreads="50"maxIdleTime="60000"/>
<Connector port="8080" protocol="HTTP/1.1"executor="tomcatThreadPool"/>
```

大致意思是原先使用传统的线程池Worker来处理Acceptor线程池中的socket连接，我们可以自定义线程池（线程池类型和连接属性）来enhance性能。



其中：

• namePrefix：线程池中线程的命名前缀

• maxThreads：线程池的最大线程数

• minSpareThreads：线程池的最小空闲线程数

• maxIdleTime：超过最小空闲线程数时，多的线程会等待这个时间长度，然后关闭

• threadPriority：线程优先级



注：当tomcat并发用户量大的时候，单个jvm进程确实可能打开过多的文件句柄，这时会报java.net.SocketException:Too many open files错误。

可使用下面步骤检查：

• ps -ef |grep tomcat 查看tomcat的进程ID，记录ID号，假设进程ID为10001

• lsof -p 10001|wc -l 查看当前进程id为10001的 文件操作数

• 使用命令：ulimit -a 查看每个用户允许打开的最大文件数



#### 2.3 Listener

还有一个影响tomcat性能的因素是内存泄漏,我们在Server标签中配置一个JreMemoryLeakPreventionListener就可以用来预防JRE内存泄漏。

作用：防止JVM出现OutOfMemory（原理是起守护线程，每隔一小时执行一次full GC）。



用法：在server.xml中配置：

```
<Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" gcDaemonProtection="false"/>
```

黑色标记部分解决了每一小时就执行一次full gc 带来的延时问题（有时heap内存未满，老年代未达到上限）



其他配置如下：

```
1.  <Executorname="tomcatThreadPool" namePrefix="catalina-exec-"   

2.  maxThreads="800"minSpareThreads="25" maxIdleTime="60000"/>  

3.  <Connectorexecutor="tomcatThreadPool"   

4.      port="80"protocol="HTTP/1.1"  

5.      connectionTimeout="60000"  

6.      keepAliveTimeout="15"  

7.      maxKeepAliveRequests="200"                    

8.      disableUploadTimeout="false"  

9.      enableLookups="false"  

10. redirectPort="8443"/> 
```



### 3. 组件优化

可以选用Tomcat Native组件，它可以让 Tomcat使用 Apache 的 APR包来处理包括文件和网络IO操作，从而提升性能及兼容性。