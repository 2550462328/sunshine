java堆在虚拟机启动的时候建立，它是java程序最主要的内存工作区域，几乎所有的java对象实例都存放在java堆中。堆空间是所有进程共享的，这是一块与java应用息息相关的内存空间。

堆也是垃圾回收重点管理的区域，从GC角度来讲，可以分为新生代和永生代，而新生代又有Eden区和Survivor区。

![img](http://pcc.huitogo.club/be576b528b68166e4f3c836e8d494c42)



#### 1. 新生代

JVM 新创建的对象（除了大对象外）会被存放在新生代，默认占 1/ 3堆内存空间。由于 JVM 会频繁创建对象，所以新生代会频繁触发 minorGC 进行垃圾回收。

新生代又分为 Eden 区、 SurvivorTo 区和 SurvivorFrom 区，如下所述

- Eden 区： Java 新创建的对象首先会被存放在Eden区 ，如果新创建的对象属于大对象， 直接将其分配到老年代。大对象的定义和具体的 JVM 版本、堆大小和垃圾回收策略有关，一般为 2KB~128KB ，可通过 XX PretenureSizeThreshold 设置其大小。在Eden 区的内存空间不足时会触发 MinorGC ，对新生代进行一次垃圾回收
- SurvivorTo ： 保留上一次 MinorGC 时的幸存者
- ServivorFrom：将上一次MinorGC 时的幸存者作为这一次MinorGC 的被扫描者。



新生代的 GC 叫作 MinorGC，采用复制算法实现，具体过程如下

- 把在 Eden 区和SurvivorFrom 区中存活的对象复制到 SurvivorTo区。如果某对象的年龄达到老年代的标准（对象晋升老年代的标准由 XX:MaxTenuringThreshold 设置，默认为 15 ）， 则将其复制到老年代， 同时把这些对象的年龄加1 ；如果 SurvivorTo 区的空间不够，则也直接将其复制到老年代； 如果对象属于大对象（大小为 2KB ~128K对象属于大对象 ，例如通过 XX:PretenureSizeThreshold =2097152 设置大对象为 2MB）， 则也直接将其复制到老年代。
- 清空 Eden和SurvivorFrom 区中的对象。
- 将 SurvivorTo 区和 SurvivorFrom 区互换，原来的 SurvivorTo 区成为下一次GC时的 SurvivorFrom区。



#### 2. 老年代

老年代主要存放有长生命周期的对象和大对象 。 老年代的 GC 过程叫作 MajorGC 。 在老年代，对象 比较稳定， MajorGC 不会被频繁触发 。 在进行 MajorGC 前， JVM 会进行 一次 MinorGC ，在 MinorGC 过后仍然 出现老年代空间不足或无法找到足够大的连续空间 分配给新创建的大对象时， 会触发 MajorGC 进行垃圾回收， 释放 JVM 的内存空间 。

MajorGC 采用标记清除算法，该算法首先会扫 描所有对象并标记存活的对象，然后 回收未被标记的对象，并释放内存空间 。

因为要先扫描老年代的所有对象再回收，所以 MajorGC 的耗时较长 。 MajorGC 的标 记清除算法容易产生内存碎片 。 在老年代没有内存空间可分配时， 会抛出 Out Of Memory 异常 。



#### 3. 永久代

永久代指内存的永久保存 区域，主要存放 Class 和 Meta （元数据）的信息 。 Class 在 类加载时被放入永久代 。 永久代和老年代、新生代不同， GC 不会在程序运行期间对永久 代的内存进行清理，这也导致了永久代的内存会随着加载的 Class 文件的增加而增加，在 加载的 Class 文件过多时会抛出 Out Of Memory 异常 ，比如 Tomcat 引用 Jar 文件过多导 致 JVM 内存不足而无法启动。

需要注意的是，在 Java 8 中永久代已经被元数据区（也叫作元空间 ）取代 。 元数据 区的作用和永久代类似， 二者最大的区别在于 ： 元数据区并没有使用虚拟机的内存， 而 是直接使用操作系统的本地内存 。 因 此 ，元空间的大小不受 JVM 内存的限制，只和操作 系统的内存有关 。

在 Java 8 中， JVM 将类的元数据放入本地内存（ Native Memory ） 中，将常量池和类 的静态变量放入 Java 堆中，这样 JVM 能够加载多少元数据信息就不再由 JVM 的最大可 用内存 （ MaxPermSize ）空间决定，而由操作系统的实际可用内存空间决定 。



在 JDK 1.8中移除整个永久代，取而代之的是一个叫元空间（Metaspace）的区域（永久代使用的是JVM的堆内存空间，而元空间使用的是物理内存，直接受到本机的物理内存限制）。

![img](http://pcc.huitogo.club/bb5a1def49917ea4e82a4121147f071a)



#### 4. 为什么要分代？

java虚拟机根据对象存活的周期不同，把堆内存划分成了几块，一般分为新生代、老年代、和永久代（堆HotSpot虚拟机而言），这就是jvm的分代策略。



**为了提高对象内存分配和垃圾回收的效率**。堆内存是jvm管理内存中最大的一块，也是垃圾回收最频繁的一块，新创建的对象在新生代中创建内存，经过几次回收仍存活的对象放在老年代中，静态对象、类信息等存放在永生代中，新生代的对象存活时间短，只需要对新生代区域中频繁进行GC，老年代对象生命周期长，内存回收效率较低，不需要频繁进行回收，永生代一般不做回收，对不同年代的特点采用合适的垃圾收集算法，大大提高了收集效率，jdk1.7之后就将永生代中的字符串常量池移除，永生代主要存放常量、类信息、静态变量等数据，与垃圾回收意义不大。



**新生代**：新生代进一次垃圾收集一般可以回收70%~90的空间，收集效率很高。

eden分配内存需要的是连续的，有可能同时为两个对象分配一块相同的内存，JVM的解决方案是为每个线程都预先分配好一块连续的内存空间并规定对象存放的位置，而如果内存空间不足会再申请多块内存空间。这个操作叫**TLAB**



HotSpot将新生代划分成三块，eden空间和survivor空间（幸存空间，分成from和to），默认比例是8:1:1，划分的目的是用复制算法来回收新生代，设置这个比例是为了更好地利用内存空间。

新生代对象在eden空间分配（大对象除外，大对象直接进入老年代），当eden没有足够内存呢分配的时候，虚拟机将发起一次Minor GC。GC进行时，eden区中所有存活的对象被复制到survivor区中的to区，而survivor中from区中对象则根据年龄值决定去向，年龄到达阀值（默认为15，新生代中对象每熬过一次垃圾回收，年龄就会加1，年龄存储在对象的hearder中）会被移向老年代，没有达到的被复制到to区，接着清空eden区和from区，新生代存活的对象都存储在to区，接着to区和from区调换角色，也就是新的from区就是上次存活的to区，总之不管怎样都保证to区在新的一轮GC时为空，GC当to区没有足够空间存放存活下来的对象时，需要依赖老年代进行分配担保，将这些对象存放在老年代中，老年代承当的角色相当于银行贷款需要担保人一样，还不起款从担保人账户里面扣。



**Jdk 1.8之后没有永生代，最多到老年代。**