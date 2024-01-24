jvm在gc的时候主要回收的区域是**堆，所以堆也叫GC堆**。因为所有new出来对象都是在堆里分配内存。

Java堆从内存回收角度来说分为新生代和老年代。再细致点就是Eden区、From Survivor、To Survivor和tentired。划分的初衷是更好的回收内存（因材施教）。

![img](http://pcc.huitogo.club/50d7291d53ffbe20be697f28398e5f31)



关于对象创建后在新生代和老年代的生命历程如下：

![img](http://pcc.huitogo.club/54f5ef1ec00bf018fcc8968b37694a80)

注：其中忽略了outofmemory heap space的情况。



总结起来下面三条：

![img](http://pcc.huitogo.club/12199be80c8756fc08fa08f730bf0348)



关于对象的内存分配中有一些问题

#### 1. 年龄上限是多少？

不少人都说默认是15，这个说法是“深入理解Java虚拟机”一书中的，如果你去Oracle的官网阅读[相关的虚拟机参数](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html)，你会发现-XX:MaxTenuringThreshold=threshold这里有个说明

*Sets the maximum tenuring threshold for use in adaptive GC sizing. The largest value is 15. The default value is 15 for the parallel (throughput) collector and 6 for the CMS collector*



**默认晋升年龄并不都是15，这个是要区分垃圾收集器的，CMS就是6。**

使用-XX:MaxTenuringThreshold可设置这个年龄上限。

但实际上设置完之后jvm也会根据情况进行调整的，Hotspot遍历所有对象时，按照年龄从小到大对其所占用的大小进行累积，当累积的某个年龄大小超过了survivor区的一半时，取这个年龄和MaxTenuringThreshold中更小的一个值，作为新的晋升年龄阈值。



动态年龄计算的代码如下

```
1.  uint ageTable::compute_tenuring_threshold(size_t survivor_capacity) {  

2.      //survivor_capacity是survivor空间的大小  

3.  size_t desired_survivor_size = (size_t)((((double) survivor_capacity)*TargetSurvivorRatio)/100);  

4.  size_t total = 0;  

5.  uint age = 1;  

6.  while (age < table_size) {  

7.   total += sizes[age];//sizes数组是每个年龄段对象大小  

8.   if (total > desired_survivor_size) break;  

9.   age++;  

10. }  

11. uint result = age < MaxTenuringThreshold ? age : MaxTenuringThreshold;  

12.     ...  

13. }  
```



但是回过头来想一下这个默认晋升年龄15怎么来的？

**因为对象头里标注年龄的空间只有4位，也就是最大只能是15**



#### 2. 对象分配规则

1. 对象优先分配在Eden区，如果Eden区没有足够的空间时，虚拟机执行一次Minor GC。
2. 大对象直接进入老年代（大对象是指需要大量连续内存空间的对象）。这样做的目的是避免在 Eden区和两个Survivor区之间发生大量的内存拷贝（新生代采用复制算法收集内存）。
3. 长期存活的对象进入老年代。虚拟机为每个对象定义了一个年龄计数器，如果对象经过了1次 Minor GC那么对象会进入Survivor区，之后每经过一次Minor GC那么对象的年龄加1，直到达 到阀值对象进入老年区。
4. 动态判断对象的年龄。如果Survivor区中相同年龄的所有对象大小的总和大于Survivor空间的 一半，年龄大于或等于该年龄的对象可以直接进入老年代。
5. 空间分配担保。每次进行Minor GC时，JVM会计算Survivor区移至老年区的对象的平均大小， 如果这个值大于老年区的剩余值大小则进行一次Full GC，如果小于检查HandlePromotionFailure设置，如果true则只进行Monitor GC,如果false则进行Full GC。



#### 3. 对象一定在堆上分配内存吗？

不一定，**逃逸分析**

逃逸分析(Escape Analysis)，是一种可以有效减少Java 程序中同步负载和内存堆分配压力的跨函数 全局数据流分析算法。通过逃逸分析，Java Hotspot编译器能够分析出一个新的对象的引用的使用 范围，从而决定是否要将这个对象分配到堆上。

逃逸分析是指分析指针动态范围的方法，它同编译器优化原理的指针分析和外形分析相关联。当变 量（或者对象）在方法中分配后，其指针有可能被返回或者被全局引用，这样就会被其他方法或者 线程所引用，这种现象称作指针（或者引用）的逃逸(Escape)。

通俗点讲，如果一个对象的指针被 多个方法或者线程引用时，那么我们就称这个对象的指针发生了逃逸。