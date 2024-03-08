#### 1. Jps

jps(JVM Process Status) 命令类似 UNIX 的 ps 命令。

- jps：显示虚拟机执行主类名称以及这些进程的本地虚拟机唯一 ID
- jps -q ：只输出进程的本地虚拟机唯一 ID。
- jps -l:输出主类的全名，如果进程执行的是 Jar 包，输出 Jar 路径。
- jps -v：输出虚拟机进程启动时 JVM 参数。
- jps -m：输出传递给 Java 进程 main() 函数的参数。



#### 2. Jstat

**监视虚拟机各种运行状态信息**

它可以显示本地或者远程（需要远程主机提供 RMI支持）虚拟机进程中的类信息、内存、垃圾收集、JIT 编译等运行数据，在没有GUI，只提供了纯文本控制台环境的服务器上，它将是运行期间定位虚拟机性能问题的首选工具。



jstat 命令使用格式：

```
jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count>]]
```



其中常见的option如下

- jstat -class vmid ：显示 ClassLoader 的相关信息；
- jstat -compiler vmid ：显示 JIT 编译的相关信息；
- jstat -gc vmid ：显示与 GC 相关的堆信息；
- jstat -gccapacity vmid ：显示各个代的容量及使用情况；
- jstat -gcnew vmid ：显示新生代信息；
- jstat -gcnewcapcacity vmid ：显示新生代大小与使用情况；
- jstat -gcold vmid ：显示老年代和永久代的行为统计，从jdk1.8开始,该选项仅表示老年代，因为永久代被移除了；
- jstat -gcoldcapacity vmid ：显示老年代的大小；
- jstat -gcpermcapacity vmid ：显示永久代大小，从jdk1.8开始,该选项不存在了，因为永久代被移除了；
- jstat -gcutil vmid ：显示垃圾收集信息；



例如jstat -gc -h3 31736 1000 10表示分析进程 id 为 31736 的 gc 情况，每隔 1000ms 打印一次记录，打印 10 次停止，每 3 行后打印指标头部。



#### 3. Jinfo

**实时地查看和调整虚拟机各项参数**

jinfo vmid：输出当前 jvm 进程的全部参数和系统属性(第一部分是系统的属性，第二部分是 JVM 的参数)。



jinfo -flag name vmid :输出对应名称的参数的具体值。

例如jinfo -flag MaxHeapSize 17340：输出 jvm进程为17340的MaxHeapSize



jinfo -flag PrintGC 17340：查看 jvm 进程是否开启打印 GC 日志



使用 jinfo 可以在不重启虚拟机的情况下，可以动态的修改 jvm 的参数。尤其在线上的环境特别有用，格式如下：

jinfo -flag [+|-]name vmid 开启或者关闭对应名称的参数。

例如jinfo -flag +PrintGC 17340：为jvm进程添加打印GC日志参数。



#### 4. Jmp

**生成堆转储快照**

如果不使用 jmap 命令，要想获取 Java 堆转储，可以使用 “-XX:+HeapDumpOnOutOfMemoryError” 参数，可以让虚拟机在 OOM异常出现之后自动生成 dump 文件，Linux 命令下可以通过 kill-3 发送进程退出信号也能拿到 dump 文件。

jmap 的作用并不仅仅是为了获取 dump 文件，它还可以查询 finalizer 执行队列、Java堆和永久代的详细信息，如空间使用率、当前使用的是哪种收集器等。



示例：将指定应用程序的堆快照输出到桌面。

jmap -dump:format=b,file=C:UsersSnailClimbDesktopheap.hprof 17340

![img](http://pcc.huitogo.club/d962c0824ecb61b35a06056d67fd0692)



使用jhat分析heap.hprof文件（或者Visual VM工具）

jhat C:UsersSnailClimbDesktopheap.hprof

![img](http://pcc.huitogo.club/bf33e793de803ae0ec5ec0dc2af6c115)

访问 http://localhost:7000/



#### 5. Jstack

生成虚拟机当前时刻的线程快照

线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈的集合

生成线程快照的目的主要是定位线程长时间出现停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间等待等都是导致线程长时间停顿的原因。线程出现停顿的时候通过jstack来查看各个线程的调用堆栈，就可以知道没有响应的线程到底在后台做些什么事情，或者在等待些什么资源。

使用上直接jstack pid



#### 6. jconsole可视化工具

基于JMX的可视化、管理工具

可以查看内存、线程、类、CPU信息，以及对JMX MBean进行管理



#### 7. visual Vm可视化工具

VisualVM（All-in-One Java Troubleshooting Tool）是到目前为止随 JDK发布的功能最强大的运行监视和故障处理程序，官方在 VisualVM的软件说明中写上了“All-in-One”的描述字样，预示着他除了运行监视、故障处理外，还提供了很多其他方面的功能，如性能分析（Profiling）。VisualVM 的性能分析功能甚至比起 JProfiler、YourKit 等专业且收费的 Profiling工具都不会逊色多少，而且 VisualVM 还有一个很大的优点：不需要被监视的程序基于特殊Agent运行，因此他对应用程序的实际性能的影响很小，使得他可以直接应用在生产环境中。这个优点是JProfiler、YourKit 等工具无法与之媲美的。



VisualVM 基于 NetBeans平台开发，因此他一开始就具备了插件扩展功能的特性，通过插件扩展支持，VisualVM可以做到：

1. 显示虚拟机进程以及进程的配置、环境信息（jps、jinfo）。
2. 监视应用程序的 CPU、GC、堆、方法区以及线程的信息（jstat、jstack）。
3. dump 以及分析堆转储快照（jmap、jhat）。
4. 方法级的程序运行性能分析，找到被调用最多、运行时间最长的方法。
5. 离线程序快照：收集程序的运行时配置、线程 dump、内存 dump等信息建立一个快照，可以将快照发送开发者处进行 Bug 反馈。
6. 其他 plugins 的无限的可能性......



**如何使用jvisualvm实现对远程项目的监控？**

核心启动项目的时候配置jmx参数，类似如下

```
1.  nohup java -jar   

2.  -Dcom.sun.management.jmxremote.port=8999   

3.  -Dcom.sun.management.jmxremote.ssl=false   

4.  -Dcom.sun.management.jmxremote.authenticate=false   

5.  -Djava.rmi.server.hostname=158.222.164.207   

6.  atom-0.0.14.jar    
```



然后在jvisualvm中选择远程 --> 添加主机 --> 添加JMX连接