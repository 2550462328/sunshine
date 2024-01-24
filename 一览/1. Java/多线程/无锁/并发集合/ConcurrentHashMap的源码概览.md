首先提出一些问题？

1）多线程操作的时候读写并发吗？

2）每次扩容大小？扩容的时候允许读写吗？

3）多线程怎么帮助扩容？

4）多线程怎么计数的？

5）无锁式实现并发怎么做的？

答案在本文中找.......

Start GO！！！



### 1、为什么会使用ConcurrentHashMap这个集合？

因为传统的hashMap在多线程并发的情况是不安全的，所以使用了concurrentHashMap代替了多线程下的hashMap



### 2、ConcurrentHashMap的成员常量

#### 2.1 限制值（用于边界值等条件判断）

```
2.  // map的最大容量  

3.  private static final int MAXIMUM_CAPACITY = 1 << 30;   

4.  // map的初始容量  

5.  private static final int DEFAULT_CAPACITY = 16;  

6.  // 虚拟机限制的最大数组长度，用在Collection.toArray()时限制大小  

7.  static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;  

8.  // map默认并发数(兼容jdk1.7)  

9.  private static final int DEFAULT_CONCURRENCY_LEVEL = 16;  

10. // 扩容因子(兼容jdk1.7),jdk1.8中使用n - ( n \>\> 2)代替 \*0.75f  

11. private static final float LOAD_FACTOR = 0.75f;  

12. // 链表转树的临界节点数  

13. static final int TREEIFY_THRESHOLD = 8;  

14. // 树转链表的临界节点数  

15. static final int UNTREEIFY_THRESHOLD = 6;  

16. // 链表转树时Node[]的长度不小于64  

17. static final int MIN_TREEIFY_CAPACITY = 64;  

18. // 每个线程帮助扩容的时候最少要处理的hash桶数，这个值如果太小会导致多线程竞争数过多  

19. // 在计算的时候 认为一个CPU可以处理8个线程的并发，所以每个线程需要处理的hash桶数是(table.length) / 8 / CPU个数 如果小于 16 就让他等于16  

20. private static final int MIN_TRANSFER_STRIDE = 16;  

21. // 每个扩容都唯一的生成戳的数，最小是6  

22. private static int RESIZE_STAMP_BITS = 16;  

23. // 最大的扩容线程的数量(2\^16 - 1)  

24. private static final int MAX_RESIZERS = (1 \<\< (32 - RESIZE_STAMP_BITS)) - 1;  

25. // 移位量，把生成戳移位后保存在sizeCtl中当做扩容线程计数的基数，向反方向移位后能够反解出生成戳  

26. private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;

27. // 可使用的CPU内核数

28. static final int NCPU = Runtime.getRuntime().availableProcessors();
```



#### 2.2 Node节点的hash值相关

```
30. // 下面是一些特殊节点的hash值，正常节点hash在spread函数都会生成正数  

31. // 这个hash值出现在扩容的时候会有一个Forwarding的临时节点，它不存储实际的数据  

32. // 如果旧数组的一个hash桶中全部的节点都迁移到新数组中，旧数组就在这个hash桶中放置一个ForwardingNode  

33. // 读操作或者迭代读时碰到ForwardingNode时，将操作转发到扩容后的新的table数组上去执行，写操作碰见它时，则尝试帮助扩容  

34. static final int MOVED = -1;   

35. // 这个hash值出现在树的头部节点TreeBin，它指向树的根节点root  

36. // TreeBin维护了一个简单读写锁  

37. static final int TREEBIN = -2;   

38. // ReservationNode的hash值，ReservationNode是一个保留节点，相当于一个预留位置，不会保存实际的数据，正常情况是不会出现的  

39. static final int RESERVED = -3;   

40. // 用于和负数hash值进行 & 运算，将其转化为正数（绝对值不相等  

41. static final int HASH_BITS = 0x7fffffff;   
```



### 3. ConcurrentHashMap的成员变量

#### 3.1 跟多线程扩容相关

```
44. // 存放node节点的数组  

45. transient volatile Node<K, V>[] table;  


46. /** 扩容后的新的table数组，只有在扩容时才有用 

47.   * nextTable != null，说明扩容方法还没有真正退出，一般可以认为是此时还有线程正在进行扩容 

48.   */  

49. private transient volatile Node<K, V>[] nextTable;  


50.    /** 

51.     * 非常重要的属性 

52.     * sizeCtl = -1，表示有线程正在进行真正的初始化操作 

53.     * sizeCtl = -(1 + nThreads)，表示有nThreads个线程正在进行扩容操作（实际上不是这么计算参与线程的） 

54.     * sizeCtl > 0，表示接下来的真正的初始化操作中使用的容量，或者初始化/扩容完成后的threshold（table.length * 3/4） 

55.     * sizeCtl = 0，默认值，此时在真正的初始化操作中使用默认容量 

56.     */  

57. private transient volatile int sizeCtl;  


58. /** 

59.  * 调整大小时要分割的下一个表索引(上一个transfer任务的起始下标index 加上1)。 

60.  * transfer时方向是从大到小的，迭代时是下标从小往大，二者方向相反，尽量减少扩容时transefer和迭代两者同时处理一个hash桶的情况 

61.  * 顺序相反时，二者相遇过后，迭代没处理的都是已经transfer的hash桶，transfer没处理的，都是已经迭代的hash桶，冲突会变少 

62.  * 下标在[nextIndex - 实际的stride （下界要 >= 0）, nextIndex - 1]内的hash桶，就是每个transfer的任务区间 

63.  * 每次接受一个transfer任务，都要CAS执行 transferIndex = transferIndex - 实际的stride，保证一个transfer任务不会被几个线程同时获取（相当于任务队列的size减1） 

64.  * 当没有线程正在执行transfer任务时，一定有transferIndex <= 0，这是判断是否需要帮助扩容的重要条件（相当于任务队列为空） 

65.  */  

66. private transient volatile int transferIndex;  
```



#### 3.2 跟高效的并发计数方式有关

```
68. // 下面三个主要与统计数目有关，可以参考jdk1.8新引入的java.util.concurrent.atomic.LongAdder的源码，帮助理解  

69. // 计数器基本值，主要在没有碰到多线程竞争时使用，需要通过CAS进行更新  

70. private transient volatile long baseCount;  


71. // CAS自旋锁标志位，用于初始化，或者counterCells扩容时  

72. private transient volatile int cellsBusy;  


73. // 用于高并发的计数单元，如果初始化了这些计数单元，那么跟table数组一样，长度必须是2^n的形式  

74. private transient volatile CounterCell[] counterCells;  
```



### 4. concurrentHashMap的构造函数

Jdk1.8的构造函数是不带loadFactor的，带loadFactor是为了向下兼容jdk1.7

```
1.  public ConcurrentHashMapDebug(int initialCapacity) {  

2.      if (initialCapacity < 0)  

3.          throw new IllegalArgumentException();  

4.      int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY  

5.              : tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));  

6.      this.sizeCtl = cap;  

7.  }  

8.  public ConcurrentHashMapDebug(Map<? extends K, ? extends V> m) {  

9.      this.sizeCtl = DEFAULT_CAPACITY;  

10.     putAll(m);  

11. }  
```

可以看到在构造函数中没有初始化table数组，将sizeCtrl值设为cap



### 5. concurrentHashMap的内部类

![img](http://pcc.huitogo.club/6dfd2b8ff985c4540a44cea5c94da2c8)



### 6. ConcurrentHashMap的cas方法

```
3. // 用来返回节点数组的指定位置的节点的原子操作  

@SuppressWarnings("unchecked")  

4.  static final <K, V> Node<K, V> tabAt(Node<K, V>[] tab, int i) {  

5.      return (Node<K, V>) U.getObjectVolatile(tab, ((long) i << ASHIFT) + ABASE);  

6.  }  


7.  // cas操作，用来在指定位置修改值  

8.  static final <K, V> boolean casTabAt(Node<K, V>[] tab, int i, Node<K, V> c, Node<K, V> v) {  

9.      return U.compareAndSwapObject(tab, ((long) i << ASHIFT) + ABASE, c, v);  

10. }  


11. // 原子操作，在指定位置设置值  

12. static final <K, V> void setTabAt(Node<K, V>[] tab, int i, Node<K, V> v) {  

13.     U.putObjectVolatile(tab, ((long) i << ASHIFT) + ABASE, v);  

14. }  
```

可以看到底层都是通过Unsafe类实现cas操作的