**首先为什么在有AtomicLong的情况下使用LongAdder计数？**

AtomicLong内部只有一个volatile long value，这种非阻塞的原子操作虽然说相对原有的阻塞算法来说已经很好了，但是在高并发下多线程同时竞争一个原子变量的更新操作，由于同一时间只会有一个线程CAS操作成功，**会造成大量的线程竞争失败后无限尝试重试**。这无疑浪费了很多开销。JDK8的LongAdder，就能解决高并发的递增或递减的原子操作。



使用AtomicLong和LongAdder的流程图大致如下：

![img](http://pcc.huitogo.club/23c6a670d08ea109223cb1efc41c3d17)



**1. 先看一下LongAdder的父类Striped64的成员变量**

```
1.  static final int NCPU = Runtime.getRuntime().availableProcessors();  

2.  transient volatile Cell[] cells;  

3.  transient volatile long base;  

4.  transient volatile int cellsBusy; 
```

1）NCPU

获取当前所有CPU的核心数总和，用于扩容的判断条件。

2）Cell[] cells

存放Cell的hash表，大小为2的幂

3）long base

基础值，在没有竞争时会更新这个值；在cells初始化的过程中，cells处于不可用的状态，这时候也会尝试将通过cas操作值累加到base



Cell[] cells 和 long base是保存数据的方法，AtomicInteger只有一个value，所有线程累加都要通过cas竞争value这一个变量，高并发下线程争用非常严重；

LongAdder则有两个值用于累加，一个是base，它的作用类似于AtomicInteger里面的value，在没有竞争的情况不会用到cells数组，它为null，这时使用base做累加，有了竞争后cells数组就上场了，第一次初始化长度为2，以后每次扩容都是变为原来的两倍，直到cells数组的长度大于等于当前服务器cpu的数量为止就不在扩容（因为在线程数大于CPU的时候会发生更多的CAS失败）；每个线程会通过线程对cells[threadLocalRandomProbe%cells.length] 位置的Cell对象中的value做累加，这样相当于将线程绑定到了cells中的某个cell对象上；

4）cellsBusy

它有两个值0 或1，它的作用是当要修改cells数组时加锁，防止多线程同时修改cells数组，0为无锁，1为加锁，加锁的状况有三种

A. cells数组初始化的时候；

B. cells数组扩容的时候；

C. 如果cells数组中某个元素为null，给这个位置创建新的Cell对象的时候；



**2. LongAdder的数据单元模块是Cell，看一下Cell**

```
1.  @sun.misc.Contended static final class Cell {  

2.          volatile long value;  

3.          Cell(long x) { value = x; }  

4.          final boolean cas(long cmp, long val) {  

5.              return UNSAFE.compareAndSwapLong(this, valueOffset, cmp, val);  

6.          }  

7.          private static final sun.misc.Unsafe UNSAFE;  

8.          private static final long valueOffset;  

9.          static {  

10.             ... // 初始化valueOffset  

11.         }  

12.     } 
```

我们可以看到每一个Cell在初始化的时候long value = 0，每一次的原子操作相当于是调用了单个数组对象的CAS，也就是说如果失败重试那么仅仅只会影响这个数组对象。

**注意的是这里@Contended用来避免CPU缓存行伪共享的问题（伪共享的详情看下面）。**



**3. LongAdder的核心方法**

**3.1 sum() 求和**

```
1.  public long sum() {  

2.      Cell[] as = cells; Cell a;  

3.      long sum = base;  

4.      if (as != null) {  

5.          for (int i = 0; i < as.length; ++i) {  

6.              if ((a = as[i]) != null)  

7.                  sum += a.value;  

8.          }  

9.      }  

10.     return sum;  

11. }  
```

可以看出LongAdder的数据和是有base数据和cell数组的数据和组成



**3.2 add(long x) 累加值**

```
1.  public void add(long x) {  

2.       Cell[] as; long b, v; int m; Cell a;  

3.       if ((as = cells) != null || !casBase(b = base, b + x)) {   //（1）和（2）

4.           boolean uncontended = true;  

5.           if (as == null || (m = as.length - 1) < 0 ||   // （3）

6.               (a = as[getProbe() & m]) == null ||   // （4）

7.               !(uncontended = a.cas(v = a.value, v + x)))   // （5）

8.               longAccumulate(x, null, uncontended);  

9.       }  

10.  }  
```

从上面代码可以看到

1. 如果cell数组不为空的话，直接cas累加base的值，这个就类似于AtomLong了。
2. 如果cell数组不为空，cas设置base值失败的话，开始计算当前线程应该访问cells中的哪个元素。
3. 如果cell数组为空，直接调用longAccumulate进行cell数组初始化，初始大小为2，后面扩容都是2的幂数。
4. 如果cells被初始化，且它的长度不为0，则通过getProbe方法获取当前线程Thread的threadLocalRandomProbe变量的值，初始为0，然后执行threadLocalRandomProbe&(cells.length-1),相当于m%cells.length;如果cells[threadLocalRandomProbe%cells.length]的位置为null，这说明这个位置从来没有线程做过累加，需要进入if继续执行，在这个位置创建一个新的Cell对象（cellBusy锁住cell数组，生成新的cell，将累积值x作为初始值）。
5. 尝试cas累积映射元素的值，如果失败（当前位置有冲突）就调用longAccumulate重新获取映射元素。



**4. Striped64的longAccumulate方法**

LongAdder的add方法适用于cell存在且更新无竞争的情况，累积在base值即可，在发生竞争时则需要执行Striped4的longAccumulate方法（初始化cell数组、获取线程探针（映射元素）、设置cell[x]的值，case累积cell的value、进行扩容、累积base值等操作）



先上源码

```
1.  final void longAccumulate(long x, LongBinaryOperator fn,  boolean wasUncontended) {   // 累积值、双目运算符、是否有竞争

3.       int h;  

4.       if ((h = getProbe()) == 0) {  

5.            // 未初始化的  

6.            ThreadLocalRandom.current();  // 强制初始化  

7.            h = getProbe();  

8.            wasUncontended = true;  

9.       }  

10.      // 最后的槽不为空则 true，也用于控制扩容，false重试。  

11.      boolean collide = false;  

12.      for (;;) {  

13.           Cell[] as; Cell a; int n; long v;  

14.           if ((as = cells) != null && (n = as.length) > 0) {  

15.                // 表已经初始化  

16.                if ((a = as[(n - 1) & h]) == null) {  

17.                     // 线程所映射到的槽是空的。  

18.                     if (cellsBusy == 0) {       // 尝试关联新的Cell  

19.                          // 锁未被使用，乐观地创建并初始化cell。  

20.                          Cell r = new Cell(x);  

21.                          if (cellsBusy == 0 && casCellsBusy()) {  

22.                               // 锁仍然是空闲的、且成功获取到锁  

23.                               boolean created = false;  

24.                               try {          // 在持有锁时再次检查槽是否空闲。  

25.                                    Cell[] rs; int m, j;  

26.                                    if ((rs = cells) != null &&  

27.                                         (m = rs.length) > 0 &&  

28.                                         rs[j = (m - 1) & h] == null) {  

29.                                         // 所映射的槽仍为空  

30.                                         rs[j] = r;         // 关联 cell 到槽  

31.                                         created = true;  

32.                                    }  

33.                               } finally {  

34.                                    cellsBusy = 0;     // 释放锁  

35.                               }  

36.                               if (created)  

37.                                    break;       // 成功创建cell并关联到槽，退出  

38.                               continue;       // 槽现在不为空了  

39.                          }  

40.                     }  

41.                     // 锁被占用了，重试  

42.                     collide = false;  

43.                }  

44.                // 槽被占用了  

45.                else if (!wasUncontended)      // 已经知道 CAS 失败  

46.                     wasUncontended = true;     // 在重散列后继续  

47.                // 在当前槽的cell上尝试更新  

48.                else if (a.cas(v = a.value, ((fn == null) ? v + x :   fn.applyAsLong(v, x))))  

50.                     break;  

51.                // 表大小达到上限或扩容了；  

52.                // 表达到上限后就不会再尝试下面if的扩容了，只会重散列，尝试其他槽  

53.                else if (n >= NCPU || cells != as)  

54.                     collide = false;       // At max size or stale  

55.                //  如果不存在冲突，则设置为存在冲突  

56.                else if (!collide)  

57.                     collide = true;  

58.                // 有竞争，需要扩容  

59.                else if (cellsBusy == 0 && casCellsBusy()) {  

60.                     // 锁空闲且成功获取到锁  

61.                     try {  

62.                          if (cells == as) {     // 距上一次检查后表没有改变，扩容：加倍  

63.                               Cell[] rs = new Cell[n << 1];  

64.                               for (int i = 0; i < n; ++i)  

65.                                    rs[i] = as[i];  

66.                               cells = rs;  

67.                          }  

68.                     } finally {  

69.                          cellsBusy = 0;    // 释放锁  

70.                     }  

71.                     collide = false;  

72.                     continue;        // 在扩容后的表上重试  

73.                }  

74.                // 没法获取锁，重散列，尝试其他槽  

75.                h = advanceProbe(h);  

76.           }  

77.           else if (cellsBusy == 0 && cells == as && casCellsBusy()) {  

78.                // 加锁的情况下初始化表  

79.                boolean init = false;  

80.                try {         // Initialize table  

81.                     if (cells == as) {  

82.                          Cell[] rs = new Cell[2];  

83.                          rs[h & 1] = new Cell(x);  

84.                          cells = rs;  

85.                          init = true;  

86.                     }  

87.                } finally {  

88.                     cellsBusy = 0;     // 释放锁  

89.                }  

90.                if (init)  

91.                     break;     // 成功初始化，已更新，跳出循环  

92.           }  

93.           else if (casBase(v = base, ((fn == null) ? v + x :   fn.applyAsLong(v, x))))  

95.                // 表未被初始化，可能正在初始化，回退使用 base。  

96.                break;       // 回退到使用 base  

97.      }  

98. }  
```



整理流程可以解释如下：

```
if 表已初始化

    if 映射到的槽是空的，加锁后再次判断，如果仍然是空的，初始化cell并关联到槽。

    else if （槽不为空）在槽上之前的CAS已经失败，重试。

    else if （槽不为空、且之前的CAS没失败，）在此槽的cell上尝试更新

    else if 表已达到容量上限或被扩容了，重试。

    else if 如果不存在冲突，则设置为存在冲突，重试。

    else if 如果成功获取到锁，则扩容。

    else 重散列，尝试其他槽。

else if 锁空闲且获取锁成功，初始化表

else if 回退 base 上更新且成功则退出

else 继续下一次循环
```



**5. 线程怎么映射位置的**

hash是LongAdder定位当前线程应该将值累加到cells数组哪个位置上的，所以hash的算法是非常重要的

java的Thread类里面有一个成员变量

```
1.  @sun.misc.Contended("tlr")  

2.    int threadLocalRandomProbe;  
```

threadLocalRandomProbe这个变量的值就是LongAdder用来hash定位Cells数组位置的，平时线程的这个变量一般用不到，它的值一直都是0。



在LongAdder的父类Striped64里通过getProbe方法获取当前线程threadLocalRandomProbe的值

```
1.  static final int getProbe() {  

2.      //PROBE是threadLocalRandomProbe变量在Thread类里面的偏移量，所以下面语句获取的就是threadLocalRandomProbe的值；  

3.      return UNSAFE.getInt(Thread.currentThread(), PROBE);  

4.  }  
```



线程对LongAdder的累加操作，在没有进入longAccumulate方法前，threadLocalRandomProbe一直都是0，当发生争用后才会进入longAccumulate方法中，进入该方法第一件事就是判断threadLocalRandomProbe是否为0，如果为0，则将其设置为**0x9e3779b9**

```
1.  int h;  

2.  if ((h = getProbe()) == 0) {  

3.      ThreadLocalRandom.current();   

4.      h = getProbe();  

5.      //设置未竞争标记为true  

6.      wasUncontended = true;  

7.  }
```

重点在这行ThreadLocalRandom.current();

```
1.  public static ThreadLocalRandom current() {  

2.      if (UNSAFE.getInt(Thread.currentThread(), PROBE) == 0)  

3.          localInit();  

4.      return instance;  

5.  }  
```



在current方法中判断如果probe的值为0，则执行locaInit()方法，将当前线程的probe设置为非0的值，该方法实现如下：

```
1.  static final void localInit() {  

2.      //private static final AtomicInteger probeGenerator = new AtomicInteger();  

3.      //private static final int PROBE_INCREMENT = 0x9e3779b9;  

4.      int p = probeGenerator.addAndGet(PROBE_INCREMENT);  

5.      //prob不能为0  

6.      int probe = (p == 0) ? 1 : p; // skip 0  

7.      long seed = mix64(seeder.getAndAdd(SEEDER_INCREMENT));  

8.      //获取当前线程  

9.      Thread t = Thread.currentThread();  

10.     UNSAFE.putLong(t, SEED, seed);  

11.     //将probe的值更新为probeGenerator的值  

12.     UNSAFE.putInt(t, PROBE, probe);  

13. } 
```



probeGenerator 是static类型的AtomicInteger类，**每执行一次localInit()方法，都会将probeGenerator累加一次0x9e3779b9这个值**，0x9e3779b9这个数字的得来是 2^32除以一个常数，这个常数就是传说中的黄金比例1.6180339887；然后将当前线程的threadLocalRandomProbe设置为probeGenerator的值，如果probeGenerator 为0，这取1



**6. 扩容之后怎么重新定位线程的位置？**

将prob的值左右移位 、异或操作三次

```
1.  static final int advanceProbe(int probe) {  

2.      probe ^= probe << 13;   // xorshift  

3.      probe ^= probe >>> 17;  

4.      probe ^= probe << 5;  

5.      UNSAFE.putInt(Thread.currentThread(), PROBE, probe);  

6.      return probe;  

7.  }  
```



**7. 总结一下Striped64的设计思路**

表的条目是 **Cell 类**，一个**填充过**（通过 sun.misc.**Contended** ）的AtomicLong 的变体，用于减少缓存竞争。填充对于多数 Atomics 是过度杀伤的，因为它们一般不规则地分布在内存里，因此彼此间不会有太多冲突。但**存在于数组的原子对象将倾向于彼此相邻地放置，因此将通常共享缓存行**（对性能有巨大的副作用），在没有这个防备下。

部分地，因为Cell相对比较大，我们避免创建它们直到需要时。**当没有竞争时，所有的更新都作用到base 字段**。根据第一次竞争（更新 base 的 CAS 失败），表被**初始化为大小2**。表的大小根据更多的竞争加倍，直到大于或等于CPU数量的最小的 2的幂。表的槽在它们需要之前保持空。

一个单独的**自旋锁（“cellsBusy”）用于初始化和resize表**，还有用新的Cell填充槽。不需要阻塞锁，当锁不可得，线程尝试其他槽（或 base）。在这些重试中，会增加竞争和减少本地性，这仍然好于其他选择。

通过 **ThreadLocalRandom 维护线程探针**字段，作为每线程的哈希码。我们让它们为 0来保持未初始化直到它们在槽 0竞争。然后初始化它们为通常不会互相冲突的值。当执行更新操作时，竞争和/或表冲突通过失败了的CAS来指示。根据冲突，如果表的大小小于容量，它的大小加倍，除非有些线程持有了锁。如果一个哈希后的槽是空的，且锁可得，创建新的Cell。否则，如果槽存在，重试CAS。**重试通过 “重散列，double hashing”来继续**，使用一个次要的哈希算法（Marsaglia XorShift）来尝试找到一个自由槽位。

**表的大小是有上限**的，因为，当线程数多于CPU数时，假如每个线程绑定到一个CPU上，存在一个完美的哈希函数映射线程到槽上，消除了冲突。当我们到达容量，我们随机改变碰撞线程的哈希码搜索这个映射。因为搜索是随机的，冲突只能通过CAS失败来知道，收敛convergence 是慢的，因为**线程通常不会一直绑定到CPU上，可能根本不会发生**。然而，尽管有这些限制，在这些案例下观察到的竞争频率显著降低。

当哈希到特定 Cell 的线程终止后，**Cell可能变为空闲的**，表加倍后导致没有线程哈希到扩展的 Cell也会出现这种情况。**我们不尝试去检测或移除这些Cell**，在实例长期运行的假设下，观察到的竞争水平将重现，所以 **Cell将最终被再次需要**。对于短期存活的实例，这没关系。