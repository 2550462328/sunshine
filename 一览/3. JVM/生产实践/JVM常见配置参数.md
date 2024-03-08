对于jvm参数指令

```
1.  -XX:+<option> 启用option，例如：-XX:+PrintGCDetails启动打印GC信息的选项，其中+号表示true，开启的意思  

2.  -XX:-<option> 不启用option，例如：-XX:-PrintGCDetails关闭启动打印GC信息的选项，其中-号表示false，关闭的意思  

3.  -XX:<option>=<number> 设定option的值为数字类型，可跟单位，例如 32k, 1024m, 2g。例如：-XX:MaxPermSize=64m  

4.  -XX:<option>=<string> 设定option的值为字符串，例如： -XX:HeapDumpPath="C:UsersDaxinDesktopjvmgcin"  
```



#### 1. 堆

##### 1.1 堆大小

```
1.  -Xms<heap size>[unit]    // 最小

2.  -Xmx<heap size>[unit]   // 最大
```

其中unit有g、m和k

建议将两个值设置成一样的，防止在堆内存发生变化的时候造成额外的资源浪费。



##### 1.2 新生代大小

有两种指定方式

```
1.  -XX:NewSize=<young size>[unit]   // 最小

2.  -XX:MaxNewSize=<young size>[unit]  // 最大
```

和

```
1. -Xmn<young size>[unit]  // 最小和最大都一样
```



GC 调优策略中很重要的一条经验总结是这样说的：

```
将新对象预留在新生代，由于 Full GC 的成本远高于 MinorGC，因此尽可能将对象分配在新生代是明智的做法，实际项目中根据 GC 日志分析新生代空间大小分配是否合理，适当通过“-Xmn”命令调节新生代大小，最大限度降低新对象直接进入老年代的情况。
```



这样设置完新生代后剩余空间是老年代

也可以直接设置新生代和老年代的比例

```
1. -XX:NewRatio=1  // 新生代和老年代大小比例为1：1 新生代占整个堆内存的1/2
```



新生代在CMS收集器下NewRatio默认是2，也就是整个堆内存的1/3

但是这是一定的吗？

在MaxNewSize和NewRatio默认配置的情况下，MaxNewSize的大小其实是在HeapSize/（NewRatio + 1）和64M * cpu数量 * 13 / 10之间取最小值；这里64M不是固定的，x86 机器为64M。



在CMS收集器中相关配置源码如下：

在第3个标记中的逻辑

![img](http://pcc.huitogo.club/9a1d6f2eafbb9eeac3bc28bfbfc902d9)

建议直接用NewSize和MaxNewSize指定，防止出现上述场景。



#### 2. 永久代

也就是方法区



Jdk1.8之前

```
1.  -XX:PermSize=N // 初始大小  

2.  -XX:MaxPermSize=N // 最大大小  
```

初始空间（默认为物理内存的1/64）和最大空间（默认为物理内存的1/4）。



Jdk1.8之后使用MetaSpace替代，MetaSpace使用的是直接内存，如果不设置是没有上限的，在不断创建类的情况下是有可能耗尽所有的系统可用内存

```
1.  -XX:MetaspaceSize=N // 初始大小  

2.  -XX:MaxMetaspaceSize=N // 最大大小 
```



#### 3. 垃圾回收

##### 3.1 选择垃圾回收器

```
1.  -XX:+UseSerialGC // 串行收集器  老年代默认SerialOld收集器

2.  -XX:+UseParallelGC // 并行收集器 （关注响应时间） 老年代默认SerialOld收集器

3.  -XX:+UseParNewGC // 并行收集器 

4.  -XX:+UseG1GC // G1收集器  

5.  -XX:+UseConcMarkSweepGC // CMS收集器 年轻代默认ParNew收集器

6. -XX:+UseParallelOldGC //老年代使用Parallel Old收集器 新生代默认Parallel Scavenge收集器
```



##### 3.2 并行收集器设置

```
1. -XX:ParallelGCThreads=n设置并行收集器收集时使用的CPU数，并行收集线程数，最后和处理器数目相等。  

2. -XX:MaxGCPauseMills=n：设置并行收集最大暂停时间，单位毫秒

3. -XX:GCTimeRatio=n：设置垃圾回收时间占用程序运行时间的百分比。公式为1/(1+n)  

4. -XX:+CMSIncrementalMode：设置为增量模式。适用于单CPU情况。  
```



##### 3.3 CMS收集器设置

因为垃圾回收器选择方面推荐年轻代ParNew+ 老年代 CMS，所以重点讲解下CMS的重要设置

```
1.  -XX:+CMSInitiatingOccupancyFraction=60：触发CMS收集器的内存比例。比如60%的意思就是说，当内存达到60%，就会开始进行CMS并发收集。  

2.  -XX:+UseCMSCompactAtFullCollection：这个前面已经提过，用于在每一次CMS收集器清理垃圾后送一次内存整理。  

3.  -XX:+CMSFullGCsBeforeCompaction=1：设置在几次CMS垃圾收集后，触发一次内存整理。 

4.  -XX:+CMSClassUnloadingEnabled：当使用CMS GC时是否启用类卸载功能，也就是是否对永生代进行一次GC

5.  -XX:+CMSParallelRemarkEnabled：是否启用并行标记，仅限于年轻代是ParNewGC 
```



##### 3.4 收集GC记录

为了严格监控应用程序的运行状况，我们应该始终检查JVM的垃圾回收性能。最简单的方法是以人类可读的格式记录GC活动。

```
1.  -XX:+PrintGC //打印GC信息

2.  -XX:+PrintGCDetails //GC时打印更多详细信息

3.  -XX:+UseGCLogFileRotation   // 滚动策略，这篇日志写满了下一个，类似s4lj中每天记录一个日志一样,须配置Xloggc

4.  -XX:NumberOfGCLogFiles=4   // 滚动GC日志文件数，默认0，不滚动

5.  -XX:GCLogFileSize=100k   // GC文件滚动大小，需配置UseGCLogFileRotation

6.  -Xloggc:/path/to/gc.log   // 记录位置

7.  -XX:+PrintTenuringDistribution // 打印存活实例年龄信息

8.  -XX:+PrintGCApplicationStoppedTime //打印应用暂停时间

9.  -XX:+PrintHeapAtGC // GC前后打印堆区使用信息
```



对于gc日志的时间戳可以选择下面

```
1.  -XX:+PrintGCTimeStamps  

2.  或  

3.  XX:+PrintGCDateStamps  
```



#### 4. 内存溢出记录和处理

在一些大型应用中，应该记录内存溢出时的记录以便于后期排查问题

```
1.  -XX:+HeapDumpOnOutOfMemoryError   // 抛出内存溢出错误时导出堆信息到指定文件

2.  -XX:HeapDumpPath=./java_pid<pid>.hprof // 快照文件路径和名称  

3.  -XX:OnOutOfMemoryError="< cmd args >;< cmd args >"   // 内存溢出后的操作指令

4.  -XX:+UseGCOverheadLimit  // 限制内存溢出前vm花费在gc上的时间
```



对于第三个内存溢出后的操作指令示例

```
1. -XX:OnOutOfMemoryError="shutdown -r" // 内存溢出后关闭，多个指令用;隔开
```



#### 5. jvm系统位数

默认选择的是os 32位，可以通过下面修改

```
1. -d 64
```



#### 6. 其他

下面是我在本地设置的jvm参数

```
1.  -server                                             ## 服务器模式  

2.  -Xms2g                                              ## 初始化堆内存大小  

3.  -Xmx2g                                              ## 堆内存最大值  

4.  -Xmn256m                                            ## 年轻代内存大小，整个JVM内存=年轻代 + 年老代 + 持久代  

5.  -Xss256k                                            ## 设置每个线程的堆栈大小  

6.  -XX:PermSize=256m                                   ## 持久代内存大小  

7.  -XX:MaxPermSize=256m                                ## 最大持久代内存大小  

8.  -XX:ReservedCodeCacheSize=256m                      ## 代码缓存，存储已编译方法生成的本地代码  

9.  -XX:+UseCodeCacheFlushing                           ## 代码缓存满时，让JVM放弃一些编译代码  

10. -XX:+DisableExplicitGC                              ## 忽略手动调用GC, System.gc()的调用就会变成一个空调用，完全不触发GC  

11. -Xnoclassgc                                         ## 禁用类的垃圾回收，性能会高一点  

12. -XX:+UseConcMarkSweepGC                             ## 并发标记清除（CMS）收集器  

13. -XX:+CMSParallelRemarkEnabled                       ## 启用并行标记，降低标记停顿  

14. -XX:+UseParNewGC                                    ## 对年轻代采用多线程并行回收，这样收得快  

15. -XX:+UseCMSCompactAtFullCollection                  ## 在FULL GC的时候对年老代的压缩，Full GC后会进行内存碎片整理，过程无法并发，空间碎片问题没有了，但提顿时间不得不变长了  

16. -XX:CMSFullGCsBeforeCompaction=3                    ## 多少次Full GC 后压缩old generation一次  

17. -XX:LargePageSizeInBytes=128m                       ## 内存页的大小  

18. -XX:+UseFastAccessorMethods                         ## 原始类型的快速优化  

19. -XX:+UseCMSInitiatingOccupancyOnly                  ## 使用设定的回收阈值(下面指定的70%)开始CMS收集,如果不指定,JVM仅在第一次使用设定值,后续则自动调整  

20. -XX:CMSInitiatingOccupancyFraction=70               ## 使用cms作为垃圾回收使用70％后开始CMS收集  

21. -XX:SoftRefLRUPolicyMSPerMB=50                      ## Soft reference清除频率，默认存活1s,设置为0就是不用就清除  

22. -XX:+AlwaysPreTouch                                 ## 强制操作系统把内存真正分配给JVM  

23. -XX:+PrintClassHistogram                            ## 按下Ctrl+Break后，打印类的信息  

24. -XX:+PrintGCDetails                                 ## 输出GC详细日志  

25. -XX:+PrintGCTimeStamps                              ## 输出GC的时间戳（以基准时间的形式）  

26. -XX:+PrintHeapAtGC                                  ## 在进行GC的前后打印出堆的信息  

27. -XX:+PrintGCApplicationConcurrentTime               ## 输出GC之间运行了多少时间  

28. -XX:+PrintTenuringDistribution                      ## 参数观察各个Age的对象总大小  

29. -XX:+PrintGCApplicationStoppedTime                  ## GC造成应用暂停的时间  

30. -Xloggc:../log/gc.log                               ## 指定GC日志文件的输出路径  

31. -ea                                                 ## 打开断言机制，jvm默认关闭  

32. -Dsun.io.useCanonCaches=false                       ## java_home没有配置，或配置错误会报异常  

33. -Dsun.awt.keepWorkingSetOnMinimize=true             ## 可以让IDEA最小化到任务栏时依然保持以占有的内存，当你重新回到IDEA，能够被快速显示，而不是由灰白的界面逐渐显现整个界面，加快回复到原界面的速度  

34. -Djava.net.preferIPv4Stack=true                     ## 让tomcat默认使用IPv4  

35. -Djdk.http.auth.tunneling.disabledSchemes=""        ## 等于Basic会禁止proxy使用用户名密码这种鉴权方式,反之空就可以使用  

36. -Djsse.enablesSNIExtension=false                    ## SNI支持，默认开启，开启会造成ssl握手警告  

37. -XX:+HeapDumpOnOutOfMemoryError                     ## 表示当JVM发生OOM时，自动生成DUMP文件  

38. -XX:HeapDumpPath=D:/data/log                        ## 表示生成DUMP文件的路径，也可以指定文件名称，如果不指定文件名，默认为：java_<pid>_<date>_<time>_heapDump.hprof。    

39. -XX:-OmitStackTraceInFastThrow                      ## 省略异常栈信息从而快速抛出,这个配置抛出这个异常非常快，不用额外分配内存，也不用爬栈,但是出问题看不到stack trace，不利于排查问题  

40. -Dfile.encoding=UTF-8  

41. -Duser.name=qhong  
```



还有一些其他没用上的

```
1.  NewRatio：3                                         ## 新生代与年老代的比例。比如为3，则新生代占堆的1/4，年老代占3/4。  

2.  SurvivorRatio：8                                    ## 新生代中调整eden区与survivor区的比例，默认为8，即eden区为80%的大小，两个survivor分别为10%的大小。   

-XX:+UsePSAdaptiveSurvivorSizePolicy          ## 参数来根据生成对象的速率动态调整eden、from和to的占比

-XX:TargetSurvivorRatio                                        ## 一个计算期望存活大小Desired survivor size的参数。默认50

3.  PretenureSizeThreshold：10m                         ## 晋升年老代的对象大小。默认为0表示没有最大值，比如设为10M，则超过10M的对象将不在eden区分配，而直接进入年老代。  

4.  MaxTenuringThreshold：15                            ## 晋升老年代的最大年龄。默认为15，比如设为10，则对象在10次普通GC后将会被放入年老代。  

5.  -XX:MaxTenuringThreshold=0                          ## 垃圾最大年龄，如果设置为0的话,则年轻代对象不经过Survivor区,直接进入年老代，该参数只有在串行GC时才有效  

6.  -XX:+HeapDumpBeforeFullGC                           ## 当JVM 执行 FullGC 前执行 dump  

7.  -XX:+HeapDumpAfterFullGC                            ## 当JVM 执行 FullGC 后执行 dump  

8.  -XX:+HeapDumpOnCtrlBreak                            ## 交互式获取dump。在控制台按下快捷键Ctrl + Break时，JVM就会转存一下堆快照  

9.  -XX:+PrintGC                                        ## 输出GC日志  

10. -verbose:gc                                         ## 同PrintGC,输出GC日志  

11. -XX:+PrintGCDateStamps                              ## 输出GC的时间戳（以日期的形式，如 2013-05-04T21:53:59.234+0800）  

12. -XX:+PrintFlagsInitial                              ## 显示所有可设置参数及默认值   

13. -enablesystemassertions                             ## 激活系统类的断言  

14. -esa                                                ## 同上  

15. -disablesystemassertions                            ## 关闭系统类的断言  

16. -dsa                                                ## 同上  

17. -XX:+ScavengeBeforeFullGC                           ## FullGC前回收年轻代内存，默认开启  

18. -XX:+CMSScavengeBeforeRemark                        ## CMS remark前回收年轻代内存      

19. -XX:+CMSIncrementalMode                             ## 采用增量式的标记方式，减少标记时应用停顿时间  

20. -XX:+CMSClassUnloadingEnabled                       ## 相对于并行收集器，CMS收集器默认不会对永久代进行垃圾回收。如果希望回收，就使用该标志，注意，即使没有设置这个标志，一旦永久代耗尽空间也会尝试进行垃圾回收，但是收集不会是并行的，而再一次进行Full GC  

21. -XX:+CMSConcurrentMTEnabled                         ## 当该标志被启用时，并发的CMS阶段将以多线程执行(因此，多个GC线程会与所有的应用程序线程并行工作)。该标志已经默认开启，如果顺序执行更好，这取决于所使用的硬件
```