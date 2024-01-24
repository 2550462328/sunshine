### 1. tryPresize(int size)

进行扩容（并非扩容的实际代码）

源码如下：

```
1.  private final void tryPresize(int size) {    

2.      int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY : tableSizeFor(size + (size >>> 1) + 1);  

3.      int sc;  

4.      while ((sc = sizeCtl) >= 0) {  // 这里一直获取最新得sizeCtl 直到“拿到锁”，compareAndSwapInt替换成功  

5.          Node<K, V>[] tab = table;  

6.          int n;  

7.          if (tab == null || (n = tab.length) == 0) {  // 出现tab为null的情况就是初始化map用的是传入一个Map,这样一上来就得扩容数组，从而tab都是空得  

8.              n = (sc > c) ? sc : c; // sc理论上是小于c的，大于c的情况仅限于上面的情况，sc = Default_Capablity 而size是传入map的size  

9.              if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {  

10.                 try {  

11.                     if (table == tab) {  

12.                         @SuppressWarnings("unchecked")  

13.                         Node<K, V>[] nt = (Node<K, V>[]) new Node<?, ?>[n];   

14.                         table = nt;  

15.                         sc = n - (n >>> 2); // sizeCtl = table.length * (3/4)  

16.                     }  

17.                 } finally {  

18.                     sizeCtl = sc;  

19.                 }  

20.             }  

21.         } else if (c <= sc || n >= MAXIMUM_CAPACITY)  

22.             break;  

23.         else if (tab == table) {  

24.             int rs = resizeStamp(n); // ???  计算本次扩容的生成戳  rs >>> RESIZE_STAMP_SHIFT 必是负数   

25.             if (sc < 0) { // 如果正在扩容Table的话，则帮助扩容  

26.                 Node<K, V>[] nt;  

27.                 if ((sc >>> 2) != rs || sc == rs + 1 || sc == rs + MAX_RESIZERS || (nt = nextTable) == null || transferIndex <= 0) // MAX_RESIZERS是最大扩容的数量  

29.                     break;  

30.                 if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) // 这里的sc是线程数 多线程帮助扩容  

31.                     transfer(tab, nt); // 将第一个参数的table中的元素，移动到第二个元素的table中去，  

32.             } else if (U.compareAndSwapInt(this, SIZECTL, sc, (rs << RESIZE_STAMP_SHIFT) + 2)) // 试着让自己成为第一个执行transfer任务的线程  

33.                 transfer(tab, null); // 当第二个参数为null的时候，会创建一个两倍大小的table  

34.         }  

35.     }  

36. }  
```



上述步骤流程如下：

1. 先判断这次扩容是不是putAll引起的，也就是table是空的，如果是的话，我扩容的大小就是大于map.size * 2 + map.size + 1的最小2的幂次数，这里map是putAll时的map参数，同时是map.size < MAXIMUM_CAPACITY(2 ^ 30)的前提下。
2. 如果不是putAll引起的，就是实打实的想扩容，那么判断下现在有没有线程正在扩容
3. 有线程扩容，就helpTransfer（帮助扩容），将tab扩容成nextTable，这里和helpTransfer代码有一点不一样的就是if中加了一个判断条件(nt = nextTable) == null ：表示整个扩容过程已经结束，或者扩容过程处于一个单线程的阶段（transfer方法中创建nextTable是由单线程完成的），此时不能帮助扩容。
4. 当前没有其他线程正在扩容，尝试将自己设置为第一个扩容的，就是设置sizeCtl的值为 (rs << RESIZE_STAMP_SHIFT) + 2，这时候nextTable是null的，所以这时候扩容的大小也就是tab.length * 2（原大小的两倍）。



### 2. transfer(Node<K, V>[] tab, Node<K, V>[] nextTab)

这个就是扩容的实质性代码了

```
1.  private final void transfer(Node<K, V>[] tab, Node<K, V>[] nextTab) { // TODO  

2.      int n = tab.length, stride;  

3.      if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)  

4.          stride = MIN_TRANSFER_STRIDE;   

5.      if (nextTab == null) { // 如果new tab为空的话 就初始化一个两倍于原table的数组  

6.          try {  

7.              @SuppressWarnings("unchecked")  

8.              Node<K, V>[] nt = (Node<K, V>[]) new Node<?, ?>[n << 1];  

9.              nextTab = nt;  

10.         } catch (Throwable ex) { //处理内存不足导致的OOM，以及table数组超过最大长度，这两种情况都实际上无法再进行扩容了  

11.             sizeCtl = Integer.MAX_VALUE;  

12.             return;  

13.         }  

14.         nextTable = nextTab;  

15.         transferIndex = n;  

16.     }  

17.     int nextn = nextTab.length;  

18.   // 转发节点，在旧数组的一个hash桶中所有节点都被迁移完后，放置在这个hash桶中，表明已经迁移完，对它的读操作会转发到新数组  

19.     ForwardingNode<K, V> fwd = new ForwardingNode<K, V>(nextTab);  

20.     boolean advance = true;  

21.     boolean finishing = false; // to ensure sweep before committing nextTab  


22.     for (int i = 0, bound = 0;;) {  

23.         Node<K, V> f;  

24.         int fh;  

25.         while (advance) {  // 这里相当于每个线程领取任务  

26.             int nextIndex, nextBound;  

27.             if (--i >= bound || finishing) // 一次transfer任务还没有执行完毕  

28.                 advance = false;  

29.             else if ((nextIndex = transferIndex) <= 0) { // transfer任务已经没有了，表明可以准备退出扩容了  

30.                 i = -1;  

31.                 advance = false;  

32.             } else if (U.compareAndSwapInt(this, TRANSFERINDEX, nextIndex,  

33.                     nextBound = (nextIndex > stride ? nextIndex - stride : 0))) { // // 尝试申请一个transfer任务  

34.                 bound = nextBound; // 申请到任务后标记自己的任务区间 bound = nextIndex - stride  

35.                 i = nextIndex - 1;  

36.                 advance = false;  

37.             }  

38.         }  


39.         if (i < 0 || i >= n || i + n >= nextn) {  // 处理扩容重叠  

40.             int sc;  

41.             if (finishing) {  

42.                 nextTable = null;  

43.                 table = nextTab;  

44.                 sizeCtl = (n << 1) - (n >>> 1);  

45.                 return;  

46.             }  

47.             if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {  

48.                 if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)  

49.                     return;  

50.                 finishing = advance = true;  

51.                 i = n; // recheck before commit  

52.             }  

53.         } else if ((f = tabAt(tab, i)) == null) //  hash桶本身为null，不用迁移，直接尝试安放一个转发节点  

54.             advance = casTabAt(tab, i, null, fwd);  

55.         else if ((fh = f.hash) == MOVED)  

56.             advance = true; // already processed  

57.         else {   

58.             synchronized (f) {    

59.                 if (tabAt(tab, i) == f) {   

60.                     Node<K, V> ln, hn;  

61.                     if (fh >= 0) { //链表处理  

62.                            /* 

63.                             * 因为n的值为数组的长度，且是power(2,x)的，所以，在&操作的结果只可能是0或者n 

64.                             * 根据这个规则 

65.                             *         0-->  放在新表的相同位置 

66.                             *         n-->  放在新表的（n+原来位置） 

67.                             *         'rehash（重新散列）'的操作跟hashMap一样 

68.                             */  

69.                         int runBit = fh & n;    

70.                         Node<K, V> lastRun = f;  

71.                          /* 

72.                             * lastRun 表示的是需要复制的最后一个节点 

73.                             * 每当新节点的hash&n -> b 发生变化的时候，就把runBit设置为这个结果b 

74.                             * 这样for循环之后，runBit的值就是最后不变的hash&n的值 

75.                             * 而lastRun的值就是最后一次导致hash&n 发生变化的节点(假设为p节点) 

76.                             * 为什么要这么做呢？因为p节点后面的节点的hash&n 值跟p节点是一样的， 

77.                             * 所以在复制到新的table的时候，它肯定还是跟p节点在同一个位置 

78.                             * 在复制完p节点之后，p节点的next节点还是指向它原来的节点，就不需要进行复制了，自己就被带过去了 

79.                             * 这也就导致了一个问题就是复制后的链表的顺序并不一定是原来的倒序 

80.                             */  

81.                         for (Node<K, V> p = f.next; p != null; p = p.next) {  

82.                             int b = p.hash & n;  

83.                             if (b != runBit) {  

84.                                 runBit = b;  

85.                                 lastRun = p;  

86.                             }  

87.                         }  

88.                         if (runBit == 0) { // (注一) 这里可以认为设置完runBit = 0之后的节点都是不用复制到新数组的  

89.                             ln = lastRun;  

90.                             hn = null;  

91.                         } else { // （注二）这里可以认为设置完runBit != 0之后的节点都是需要复制到新数组的  

92.                             hn = lastRun;  

93.                             ln = null;  

94.                         }  

95.                         for (Node<K, V> p = f; p != lastRun; p = p.next) { // 这里轮询链表节点p，根据(ph & n) == 0判断是否要复制  

96.                             int ph = p.hash;  

97.                             K pk = p.key;  

98.                             V pv = p.val;  

99.                             if ((ph & n) == 0)  

100.                                 ln = new Node<K, V>(ph, pk, pv, ln); // 将不需要复制的节点放在上述(注一) lastRun的前面，如果为空相当于重新构建一个链表  

101.                             else  

102.                                 hn = new Node<K, V>(ph, pk, pv, hn); //将需要复制的节点放在上述(注二) lastRun的前面，如果为空相当于重新构建一个链表  

103.                         }  

104.                         setTabAt(nextTab, i, ln); // 将ln所在的链表放置在原地  

105.                         setTabAt(nextTab, i + n, hn); // 将hn所在的链表放在在原地 + n的位置  

106.                         setTabAt(tab, i, fwd); // 在旧tab的位置 放置 ForwardingNode节点  

107.                         advance = true; //进入下一个循环  

108.                     } else if (f instanceof TreeBin) { // 红黑树处理  

109.                         TreeBin<K, V> t = (TreeBin<K, V>) f;  

110.                         TreeNode<K, V> lo = null, loTail = null; //  

111.                         TreeNode<K, V> hi = null, hiTail = null;  

112.                         int lc = 0, hc = 0;  

113.                         for (Node<K, V> e = t.first; e != null; e = e.next) {  

114.                             int h = e.hash;  

115.                             TreeNode<K, V> p = new TreeNode<K, V>(h, e.key, e.val, null, null);  

116.                             if ((h & n) == 0) { // 将不需要复制的节点 整合到 lo ~ loTail上  

117.                                 if ((p.prev = loTail) == null)  

118.                                     lo = p;  

119.                                 else  

120.                                     loTail.next = p;  

121.                                 loTail = p;  

122.                                 ++lc;  

123.                             } else { // 将需要复制的节点 整合到 hi ~ hiTail上  

124.                                 if ((p.prev = hiTail) == null)  

125.                                     hi = p;  

126.                                 else  

127.                                     hiTail.next = p;  

128.                                 hiTail = p;  

129.                                 ++hc;  

130.                             }  

131.                         }  


132.                         // 判断新生成的两个红黑树 是否要转成链表  

133.                         ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) : (hc != 0) ? new TreeBin<K, V>(lo) : t;  

134.                         hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) : (lc != 0) ? new TreeBin<K, V>(hi) : t;  

135.                         setTabAt(nextTab, i, ln);  

136.                         setTabAt(nextTab, i + n, hn);  

137.                         setTabAt(tab, i, fwd);  

138.                         advance = true;  

139.                     }  

140.                 }  

141.             }  

142.         }  

143.     }  
```



上述扩容的步骤可以解释如下：

1）先计算每个线程需要处理的桶数（这里叫stride）

2）判断新的table是否为空，如果为空就整一个原来tab两倍的容器。

3）再进入循环，每个线程申领自己的任务，（transferIndex~transferIndex-stride）之间的桶数就是该线程的转移任务。每完成一个桶判断一下任务完成没有（是否到达了bound），如果完成了是否还能再申领任务，需要注意的是这里**处理桶数是从后往前进行处理**，迭代遍历是从前往后遍历，可以有效解决扩容时进行迭代引起的冲撞，如果冲撞了就找到放置在旧tab的ForwardingNode节点，通过它找到newTable。

4）领完任务之后需要预防一下**扩容重叠**的问题，简单的来解释就是线程A正在将数组从n->2n进行扩容（在处理中），线程B也在将数组从n->2n进行扩容，然后线程B成功扩容完了结束线程。这时候线程C要将数组从2n->4n扩容，然后线程A扩容完了，想要再领任务，这时候领的任务是从2n->4n了，但是线程A中的还是n->2n得目标，所以扩容重叠，具体可以看下面得解释。

5）线程处理任务之前需要思考一下，先是如果正在处理的桶中没有数据，直接在这个地方放一个ForwardingNode节点，然后就是这个桶是不是被别人处理过了，这时候转到下一个桶。



6）下面是线程处理任务的正式环节了，需要分别处理链表和红黑树的情况

A. 桶中的是链表，这时候仍然利用好hashMap中的优良写法，直接key.hash & tab.length == 0判断Node需不需要搬家（搬到tab.length + 当前位置）

这里有一个优化的地方就是我假设链表的后面一串都是要复制的，或者都不需要复制的，那么我就不需要大张旗鼓的一个个处理后面这些了，直接找到他们的头，让它加入我们新构建的链表，这样它的小弟就屁颠屁颠的过来了（代码中的runBit就是那个分界线）。然后在newTable放置好新构建的需要复制的链表和不需要复制的链表就阔以咯。

B. 桶中的红黑树，对待红黑树就没有上面那个骚操作了，老老实实的构建需要复制的红黑树和不需要复制的红黑树，然后放置进newTable。这里需要注意的就是红黑树需不要降解成链表的问题。



构建的过程，如图：

![img](http://pcc.huitogo.club/6ff664fd81123ac93d7abf585ddfd318)