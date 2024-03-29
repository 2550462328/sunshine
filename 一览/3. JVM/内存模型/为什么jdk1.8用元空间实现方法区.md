Jdk1.8中jvm作出了以下修改

**用元空间代替方法区**

HotSpot虚拟机已经没有PermGen space（永久代）这个区域了，取而代之的是一个叫做Metaspace（元空间）。



这里讲一下“永久代”的迭代历史

‘永久代’是方法区的实现，所以它里面存放的是**加载的类信息、常量、静态变量、即时编译器编译后的代码**等数据。

- 在jdk1.6的时候，这个PermGen space是存在的，在频繁动态生成类的情况下是会造成永久代的内存溢出。
- 在jdk1.7的时候，就开始了移除“永久代”的工作，比如将运行时常量池转移到了java heap；类的静态变量(class statics)转移到了java heap。
- 在jdk1.8的时候就已经彻底移除PermGen space，所以jdk1.8的jvm就不要配置PermSize和MaxPermSize了。取而代之的是Metaspace元空间。



**什么是元空间？**

元空间的本质和永久代类似，都是对JVM规范中方法区的实现。不过元空间与永久代之间最大的区别在于：元空间并不在虚拟机中，而是使用本地内存。因此，默认情况下，元空间的大小仅受本地内存限制，可以在jvm配置MetaspaceSize和MaxMetaspaceSize（默认是没有限制的）。



**为什么使用元空间代替方法区？**

1. 字符串常量存在方法区中，容易出现性能问题和内存溢出。

2. 类和方法的信息等比较难确定大小，因此方法区大小的指定比较困难，太小容易出现方法区溢出，太大容易导致堆空间不足。

3. 方法区的垃圾回收会带来不必要的复杂度，并且回收率偏低。

4. Oracle 可能会将HotSpot 与 JRockit 合二为一。

   参照 JEP122 ：http://openjdk.java.net/jeps/122 ，原文截取：

   > Motivation
   >
   > This is part of the JRockit and Hotspot convergence effort. JRockit customers do not need to configure the permanent generation (since JRockit does not have a permanent generation) and are accustomed to not configuring the permanent generation.

   即：移除永久代是为融合 HotSpot JVM 与 JRockit VM 而做出的努力，因为 JRockit 没有永久代，不需要配置永久代。



总之还是之前永久代的内存问题导致的，它毕竟属于堆的一部分，却承担了那么大的压力，而且堆还很嫌弃它（扫不到什么垃圾还不得不扫它），所以直接用一块直接内存去存放那些庞大的类和方法信息吧。彻底**解放堆**。