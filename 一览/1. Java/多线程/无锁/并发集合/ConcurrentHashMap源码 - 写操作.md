为什么单独讲这个方法呢，因为这个方法算是concurrentHashMap的一个核心，多线程情况下对于concurrentHashMap来说主要的竞争就是写写竞争，还包括单线程初始化table、多线程帮助扩容、并发计数等核心操作。



除了一些核心操作外，跟HashMap还是类似的（链表和红黑树的操作），可以参考HashMap的源码来学习。

putVal的源码如下：

```
1.  final V putVal(K key, V value, boolean onlyIfAbsent) {   

2.      if (key == null || value == null)  

3.          throw new NullPointerException();  

4.      int hash = spread(key.hashCode());  // 计算key的hash值  

5.      int binCount = 0; // 单个链表上元素的个数  

6.      for (Node<K, V>[] tab = table;;) {   

7.          Node<K, V> f; // 计算key后下标的Node  

8.          int n, i, fh;  // n是table长度  i是key所在链表在node数组中的下标  fh是Node的hash值  

9.          if (tab == null || (n = tab.length) == 0) // 如果node数组为空，初始化数组  

10.             tab = initTable();   

11.         else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) { // 如果计算的Key所在的位置为空，直接添加  

12.             if (casTabAt(tab, i, null, new Node<K, V>(hash, key, value, null)))   

13.                 break;   

14.         } else if ((fh = f.hash) == MOVED) // 如果检测到某个节点的hash值是MOVED，则表示正在进行数组扩张的数据复制阶段  

15.             tab = helpTransfer(tab, f);  // 则当前线程也会参与去复制，通过允许多线程复制的功能，一次来减少数组的复制所带来的性能损失  

16.         else { // 如果计算的key所在的位置有值的话  

17.             V oldVal = null;  

18.             synchronized (f) { // 给这个Node加锁  

19.                 if (tabAt(tab, i) == f) { // 再次确认之前计算下标取出的Node还是那个位置  

20.                     if (fh >= 0) { // fh >= 0 时 Node所在的位置是一个链表，fh = -2时是一个树  

21.                         binCount = 1; //自身一个Node，所以所在链表初始长度为1  

22.                         for (Node<K, V> e = f;; ++binCount) { // 遍历链表  

23.                             K ek;  

24.                             if (e.hash == hash && ((ek = e.key) == key || (ek != null && key.equals(ek)))) { // 如果取出的Node刚好Key就是要新增的key  

25.                                 oldVal = e.val;  

26.                                 if (!onlyIfAbsent) // 没有存在不可替换选项时，覆盖旧value  

27.                                     e.val = value;  

28.                                 break;  

29.                             }  

30.                             Node<K, V> pred = e;  

31.                             if ((e = e.next) == null) { // 如果取出的Node的Key不是要新增的key  

32.                                 pred.next = new Node<K, V>(hash, key, value, null); // 将新增的node放到尾部  

33.                                 break;  

34.                             }  

35.                         }  

36.                     } else if (f instanceof TreeBin) { // Node所在的位置是一棵树  

37.                         Node<K, V> p;  

38.                         binCount = 2;  // 树的头节点+自身，所以所在链表初始长度为2  

39.                         if ((p = ((TreeBin<K, V>) f).putTreeVal(hash, key, value)) != null) { // 将新节点添加到树中  

40.                             oldVal = p.val;  

41.                             if (!onlyIfAbsent)  

42.                                 p.val = value;  

43.                         }  

44.                     }  

45.                 }  

46.             }  

47.             if (binCount != 0) {  

48.                 if (binCount >= TREEIFY_THRESHOLD) // 判断是要转换成树还是扩容数组  

49.                     treeifyBin(tab, i);  

50.                 if (oldVal != null)  

51.                     return oldVal; // 返回的替换前的旧值  

52.                 break;  

53.             }  

54.         }  

55.     }  

56.     addCount(1L, binCount); // 并发计数  

57.     return null;  

58. }  
```



上面代码分成下面步骤

1. table未初始化，进行单线程初始化
2. table初始化了，经计算放置Node的下标处为null，直接cas放置值
3. table初始化了，经计算放置Node的下标处有节点，但节点的hash值为MOVED（表示table数组正在扩容），那么此时放弃添加，帮助扩容（到下个循环在添加）。
4. table初始化了，节点的hash值不为MOVED，这个时候就把存在的节点synchronized，判断是链表还是树（进行添加）。
5. 判断链表上的个数，如果超过8 是扩容还是转红黑树。
6. 因为添加了一个值，就要进行计数（+1）。



看完步骤再细看方法

#### 1.initTable()

初始化数组，保证单线程进行

```
1.  /** 

2.     * 初始化数组table， 

3.     * 如果sizeCtl小于0，说明别的数组正在进行初始化，则让出执行权 

4.     * 如果sizeCtl大于0的话，则初始化一个大小为sizeCtl的数组 

5.     * 否则的话初始化一个默认大小(16)的数组 

6.     * 然后设置sizeCtl的值为数组长度的3/4 

7.     */  

8.  private final Node<K, V>[] initTable() {  

9.      Node<K, V>[] tab;  

10.     int sc;  

11.     while ((tab = table) == null || tab.length == 0) {  

12.         if ((sc = sizeCtl) < 0) //小于0的时候表示在别的线程在初始化表或扩展表，要保证单线程扩容，进行线程礼让  

13.             Thread.yield();   

14.         else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) { //尝试cas修改sizeCtl(类似加锁) 设定为-1表示要初始化表了  

15.             try {  

16.                 if ((tab = table) == null || tab.length == 0) {  

17.                     int n = (sc > 0) ? sc : DEFAULT_CAPACITY;  

18.                     @SuppressWarnings("unchecked")  

19.                     Node<K, V>[] nt = (Node<K, V>[]) new Node<?, ?>[n];  

20.                     table = tab = nt;  

21.                     sc = n - (n >>> 2);  //初始化后，sizeCtl长度为数组长度的3/4  

22.                 }  

23.             } finally {  

24.                 sizeCtl = sc;   

25.             }  

26.             break;  

27.         }  

28.     }  

29.     return tab;  

30. }  
```



#### 2. spread(key.hashCode())

计算key的hash散列值

```
1.  static final int spread(int h) {  

2.      return (h ^ (h >>> 16)) & HASH_BITS;  // HASH_BITS =  1111111111111111111111111111111

3.  }
```

这里将（h >>> 16）^ h就是将key的hash值的高16位跟低16位进行异或，跟hashMap中操作一样，但是这里多了一个逻辑与运算。这里是**为了防止前面的异或操作出现负数的情况**，保证spread方法的返回值是一个正数。试想如果异或结果是-1或者-2的话不就跟特殊Node的Hash值冲突了么？



#### 3. helpTransfer(Node<K, V>[] tab, Node<K, V> f)

多线程帮助扩容

```
1.  final Node<K, V>[] helpTransfer(Node<K, V>[] tab, Node<K, V> f) {  

2.      Node<K, V>[] nextTab;  

3.      int sc;  

4.      if (tab != null && (f instanceof ForwardingNode) && (nextTab = ((ForwardingNode<K, V>) f).nextTable) != null) {  

5.          int rs = resizeStamp(tab.length); // 通过resizeStamp生成唯一戳，来保证这次扩容的唯一性，防止出现扩容重叠的现象  

6.          while (nextTab == nextTable && table == tab && (sc = sizeCtl) < 0) {  

7.              if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 || sc == rs + MAX_RESIZERS || transferIndex <= 0)  

8.                  break;  

9.              if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {  

10.                 transfer(tab, nextTab); // 将tab扩容到nextTab  

11.                 break;  

12.             }  

13.         }  

14.         return nextTab;  

15.     }  

16.     return table;  

17. }  
```



**这里需要先搞清楚的是rs是什么？sc又是什么？**

**rs = resizeStamp(tab.length);**

```
1.  /** 

2.   * 返回与扩容有关的一个生成戳rs，每次新的扩容，都有一个不同的n，这个生成戳就是根据n来计算出来的一个数字，n不同，这个数字也不同 

3.   * 保证 rs << RESIZE_STAMP_SHIFT（16） 必须是负数 

4.   * numberOfLeadingZeros方法返回无符号整型i的最高非零位前面的0的个数，包括符号位在内，如果是负数直接返回0 

5.   */  

6.  static final int resizeStamp(int n) {  

7.      return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));  

8.  }  
```



这里1 << (RESIZE_STAMP_BITS - 1)就是2^15 (1000 0000 0000 0000)

Integer.numberOfLeadingZeros(n) 表示int（32位）类型n的最高非零位前面的0的个数

![img](http://pcc.huitogo.club/b876356bfc821fa6cd7577f9bd5f7269)

两者进行逻辑或操作的话，会让Integer.numberOfLeadingZeros(n)结果的第16个位置为1，当n为32的时候，rs就是1000 0000 0001 1010 （32794）

如果将rs << RESIZE_STAMP_BITS（16）的话，即第32位为1（符号位），所以必是负数



**sc就是sizeCtrl**

源码注释解释说：sizeCtl = -(1 + nThreads)，表示有nThreads个线程正在进行扩容操作（实际上不是这么计算参与线程的）

在tryPresize中，第一条扩容线程会将sizeCtl变成(rs << RESIZE_STAMP_SHIFT) + 2)，sizeCtl就是在这个时候变成负数（除了初始化table的时候变成 - 1除外）。

```
U. compareAndSwapInt(this, SIZECTL, sc, (rs << RESIZE_STAMP_SHIFT) + 2)
```



上面说了rs<<16后一定是一个负数，sizeCtrl此时一定是一个负数，所以

A. sizeCtrl的高16位就是rs

B. sizeCtrl的低16位就是扩容的线程数 + 1，上面 + 2 表示此时有一个线程在扩容



这个时候就可以过来理解helpfer的源码了：

1）判断当前table正在扩容（ForwardindNode只在扩容的时候出现）。

2）再判断当前是多线程在扩容中（sizeCtl < 0）

3）在帮助扩容之前还有几个前提条件，也就是如何理解这4个if

　A. **（sc>>>RESIZE_STAMP_SHIFT） != rs**：sc >>>16 就是rs了，如果两者不相等，说明底层数组的长度改变了，说明扩容结束了，所以直接跳出循环，不在需要进行协助扩容了

　B. **transferIndex <= 0**：表示所有的transfer任务都被领取光了，没有剩余的hash桶给自己这个线程来transfer，此时线程不能再帮助扩容了

　C. **sc == rs + 1**：表示扩容线程已经达到最大值，在ConcurrentHashMap永远不相等。

　D. **sc == rs + MAX_RESIZERS**：表示扩容线程已经达到最大值，在ConcurrentHashMap永远不相等

4）4个if条件后就进入扩容环节了（领取任务），当然前提要将sc + 1（多了一个线程参与扩容）。



#### treeifyBin(Node<K, V>[] tab, int index)

将tab扩容或者在index处的链表转红黑树

```
1.  private final void treeifyBin(Node<K, V>[] tab, int index) {  //TODO  

2.      Node<K, V> b;  

3.      @SuppressWarnings("unused")  

4.      int n, sc;  

5.      if (tab != null) {  

6.          if ((n = tab.length) < MIN_TREEIFY_CAPACITY)  

7.              tryPresize(n << 1);   // 扩容  

8.          else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {  

9.              synchronized (b) {  

10.                 if (tabAt(tab, index) == b) {  

11.                     TreeNode<K, V> hd = null, tl = null; // hd 头节点 tl 尾节点  

12.                     for (Node<K, V> e = b; e != null; e = e.next) {   

13.                         TreeNode<K, V> p = new TreeNode<K, V>(e.hash, e.key, e.val, null, null);  

14.                         if ((p.prev = tl) == null)  

15.                             hd = p;  

16.                         else  

17.                             tl.next = p;  

18.                         tl = p;  

19.                     }  

20.                     setTabAt(tab, index, new TreeBin<K, V>(hd)); //将链表转换后的红黑树的头节点放在table的index所在的位置  

21.                 }  

22.             }  

23.         }  

24.     }  

25. }  
```



这里的tryPresize(n << 1) 扩容方法下面再讲，判断链表转红黑树的时候需要知道Node数组有个桶是红黑树的话，这个桶在Node数组上的头节点不是root节点，而是一个TreeBin。

```
setTabAt(tab, index, new TreeBin<K, V>(hd))
```

这里TreeBin + TreeNode 的功能相当于hashMap中的TreeNode，TreeBin指向root节点



**这里为什么用TreeBin呢？**

TreeBin的源码：

```
1.  static final class TreeBin<K, V> extends Node<K, V> {  

2.      TreeNode<K, V> root;  

3.      volatile TreeNode<K, V> first;  

4.      volatile Thread waiter;  

5.      volatile int lockState;  

6.      static final int WRITER = 1; // 写锁  

7.      static final int WAITER = 2; // 等待中  

8.      static final int READER = 4; // 读写锁  

9.      TreeBin(TreeNode<K, V> b) {  

10.         super(TREEBIN, null, null, null);  

11.         // ...  

12.     }  


13.     private final void lockRoot() {  

14.         if (!U.compareAndSwapInt(this, LOCKSTATE, 0, WRITER))  

15.             contendedLock(); // offload to separate method  

16.     }  


17.     private final void unlockRoot() {  

18.         lockState = 0;  

19.     }  


20.     private final void contendedLock() {  

21.         boolean waiting = false;  

22.         for (int s;;) {  

23.             if (((s = lockState) & ~WAITER) == 0) {  

24.                 if (U.compareAndSwapInt(this, LOCKSTATE, s, WRITER)) {  

25.                     if (waiting)  

26.                         waiter = null;  

27.                     return;  

28.                 }  

29.             } else if ((s & WAITER) == 0) {  

30.                 if (U.compareAndSwapInt(this, LOCKSTATE, s, s | WAITER)) {  

31.                     waiting = true;  

32.                     waiter = Thread.currentThread();  

33.                 }  

34.             } else if (waiting)  

35.                 LockSupport.park(this);  
```

从上面大概源码中我们知道这里是利用**TreeBin维护了一个简单的读写锁**，在putTreeVal和removeTreeNode方法中使用到。