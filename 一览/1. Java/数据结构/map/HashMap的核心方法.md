#### 1. HashMap概述

**javaDoc中描述**

在jdk1.8的javadoc中描述如下：

1. 哈希表基于map接口的实现，这个实现提供了map所有的操作，并且提供了key和value可以为null，(HashMap和HashTable大致上是一样的除了hashmap是异步的和允许key和value为null)，这个类不确定map中元素的位置，特别要提的是，这个类也不确定元素的位置随着时间会不会保持不变。
2. 假设哈希函数将元素合适的分到了每个桶(其实就是指的数组中位置上的链表)中，则这个实现为基本的操作(get、put)提供了稳定的性能，**迭代这个集合视图需要的时间跟hashMap实例(key-value映射的数量)的容量(在桶中)成正比**，因此，如果迭代的性能很重要的话，就不要将初始容量设置的太高或者loadfactor设置的太低。
3. HashMap的实例有两个参数影响性能，初始化容量(initialCapacity)和loadFactor加载因子，在哈希表中这个容量是桶的数量【也就是数组的长度】，一个初始化容量仅仅是在哈希表被创建时容量，在容量自动增长之前加载因子是衡量哈希表被允许达到的多少的。当entry的数量在哈希表中超过了加载因子乘以当前的容量，那么哈希表被修改(内部的数据结构会被重新建立)以至于哈希表有大约会增长到两倍
4. 通常来讲，默认的加载因子(0.75)能够在时间和空间上提供一个好的平衡，**更高的值会减少空间上的开支但是会增加查询花费的时间**（体现在HashMap类中get、put方法上），当设置初始化容量时，应该考虑到map中会存放 entry的数量和加载因子，以便最少次数的进行rehash操作，如果初始容量大于最大条目数除以加载因子，则不会发生rehash 操作。
5. 如果很多映射关系要存储在 HashMap 实例中，则相对于按需执行自动的 rehash操作以增大表的容量来说，使用**足够大的初始容量创建它将使得映射关系能更有效地存储**。



#### 2. hashMap数据结构和存储原理

##### 2.1 Jdk1.8之前，链表散列

链表散列 = 数组 + 链表

![img](http://pcc.huitogo.club/f1f5d774dd6f42b884e89822779701b7)

存放过程是：通过entry对象中的hash值来确定将该对象存放在数组中的哪个位置上，如果在这个位置上还有其他元素，则通过链表来存储这个元素。



Entry对象：

```
1.  static class Entry<K,V> implements Map.Entry<K,V> {  

2.  　　　  final K key;    // 键  

3.  　　　　 V value;    // 值  

4.  　　　　 Entry<K,V> next;//指向下一个entry对象  

5.  　　　　 int hash; //通过key算过来的你hashcode值。 
```



##### 2. 2 Jdk1.8之后，增加了红黑树

![img](http://pcc.huitogo.club/91de4a309058ef205c721bdee92a7fdd)

此时数据结构是 数组 + 链表 + RBT（红黑树）

在桶中数据上升到8时，链表会变成RBT，在桶中数据下降到6时，RBT会变成链表



#### 3. HashMap属性

```
3.  /** 

4.    * 数组的默认初始长度，java规定hashMap的数组长度必须是2的次方 

5.    * 扩展长度时也是当前长度 << 1。 

6.    */  

7.  static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16  

8.  // 数组的最大长度  

9.  static final int MAXIMUM_CAPACITY = 1 << 30;  

10. // 默认负载因子，当元素个数超过这个比例则会执行数组扩充操作。  

11. static final float DEFAULT_LOAD_FACTOR = 0.75f;  

12. // 树形化阈值，当链表节点个大于等于TREEIFY_THRESHOLD - 1时，  

13. // 会将该链表换成红黑树。  

14. static final int TREEIFY_THRESHOLD = 8;  

15. // 解除树形化阈值，当链表节点小于等于这个值时，会将红黑树转换成普通的链表。  

16. static final int **UNTREEIFY_THRESHOLD** = 6;  

17. // 最小树形化的容量，即：当内部数组长度小于64时，不会将链表转化成红黑树，而是优先扩充数组。  

18. static final int MIN_TREEIFY_CAPACITY = 64;  

19. // 这个就是hashMap的内部数组了，而Node则是链表节点对象。  

20. transient Node<K,V>[] table;  

21. // 下面三个容器类成员，作用相同，实际类型为HashMap的内部类KeySet、Values、EntrySet。  

22. // 他们的作用并不是缓存所有的key或者所有的value，内部并没有持有任何元素。  

23. // 而是通过他们内部定义的方法，从三个角度（视图）操作HashMap，更加方便的迭代。  

24. // 关注点分别是键，值，映射。  

25. transient Set<K>        keySet;  // AbstractMap的成员  

26. transient Collection<V> values; // AbstractMap的成员  

27. transient Set<Map.Entry<K,V>> entrySet;  

28. // 元素个数，注意和内部数组长度区分开来。  

29. transient int size;  

30. // 容器结构的修改次数，fail-fast机制。  

31. transient int modCount;  

32. // 阈值，超过这个值时扩充数组。 threshold = capacity * load factor，具体看上面的静态常量。  

33. int threshold;  

34. // 装栽因子，具体看上面的静态常量。  

35. final float loadFactor;  
```



显然，HashMap的底层实现是基于一个Node的数组，Node实现的是Map的内部接口Entry

```
1.  static class Node<K,V> implements Map.Entry<K,V> {  

2.  　　final int hash;  

3.  　　final K key;  

4.      V value;  

5.      Node<K,V> next;  

6.   }  
```



在链表变成红黑树的时候,Node也会变成TreeNode，TreeNode是Node的孙子类

继承关系是: Node是单向链表节点，Entry是双向链表节点，TreeNode是红黑树节点。

```
1.  static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {  

2.      TreeNode<K,V> parent;  // red-black tree links  

3.      TreeNode<K,V> left;  

4.      TreeNode<K,V> right;  

5.      TreeNode<K,V> prev;    // needed to unlink next upon deletion  

6.  }  
```



**Q1：Hashmap的变量都是trasient的，那么它是怎么序列化的？**

HashMap内有两个用于序列化的函数 readObject(ObjectInputStream s) 和writeObject（ObjectOutputStreams），通过这个函数将table序列化。



#### 4. HashMap构造方法

hashMap中table数组一开始就已经是个没有长度的数组了，所以**构造方法中，并没有初始化数组的大小，数组在一开始就已经被创建了**，构造方法只做两件事情，一个是**初始化加载因子**，初始化扩充阀值。

##### 4.1 HashMap()

```
1.  public HashMap() {  

2.      this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR);  

3.  } 
```



##### 4.2 HashMap(int)

```
1.  public HashMap(int initialCapacity) {  

2.      this(initialCapacity, DEFAULT_LOAD_FACTOR);  

3.  }  
```



##### 4.3 HashMap(int,float)

```
1.  public HashMap(int initialCapacity, float loadFactor) {  

2.      // 初始参数边界检查  

3.      if (initialCapacity < 0)  

4.          throw new IllegalArgumentException("Illegal initial capacity: " +  initialCapacity);  

5.      if (initialCapacity > MAXIMUM_CAPACITY)  

6.          initialCapacity = MAXIMUM_CAPACITY;  

7.      if (loadFactor <= 0 || Float.isNaN(loadFactor))  

8.          throw new IllegalArgumentException("Illegal load factor: " +  loadFactor);                                          

10.     this.loadFactor = loadFactor;  

11.     // 初始化threshold大小  

12.     this.threshold =  tableSizeFor(initialCapacity);      

13. }  
```



tableSizeFor(initialCapacity)：数组容量必须是2的次方，所以就需要通过某个算法将我们给的数值转换成2的次方。

```
1.  //  Returns a power of two size for the given target capacity.  

2.  // 返回给定目标容量的二次幂，这个方法可以将任意一个整数转换成2的次方。   

3.  static final int tableSizeFor(int cap) {  

4.      int n = cap - 1;  

5.      n |= n >>> 1;  

6.      n |= n >>> 2;  

7.      n |= n >>> 4;  

8.      n |= n >>> 8;  

9.      n |= n >>> 16;  

10.     return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;  

11. }  
```

原理实际上就是补位，将原本为0的空位填补为1，最后加1时，最高有效位进1，其余变为0。 例如1010 0101最后会变成1111 1111，然后+1变成了 1 0000 0000，为2的8次方，上面代码中int n = cap - 1先减去1就是为了最后面这个+1操作



##### 4.4 HashMap(Map<? extends K, ? extends V> m)

```
1.  public HashMap(Map<? extends K, ? extends V> m) {  

2.      // 初始化填充因子  

3.      this.loadFactor = DEFAULT_LOAD_FACTOR;  

4.      // 将m中的所有元素添加至HashMap中  

5.      putMapEntries(m, false);  

6.  } 
```



putMapEntries(m, false)：将m的所有元素存入本HashMap实例中

```
1.  final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {  

2.      int s = m.size();  

3.      if (s > 0) {  

4.          // 判断table是否已经初始化  

5.          if (table == null) { //未初始化  

6.              // 这里临时将集合m的容量作为扩容阀值 计算 table的capability  

7.              float ft = ((float)s / loadFactor) + 1.0F;  

8.              int t = ((ft < (float)MAXIMUM_CAPACITY) ? (int)ft : MAXIMUM_CAPACITY);  

10.             // 这里threshold为0 可以理解成初始化扩容阀值  

11.             if (t > threshold)  

12.                 threshold = tableSizeFor(t);  

13.         }  

14.         // 已初始化，并且m元素个数大于扩容阈值，进行扩容处理  

15.         else if (s > threshold)  

16.             resize();   //***

17.         // 将m中的所有元素添加至HashMap中  

18.         for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {  

19.             K key = e.getKey();  

20.             V value = e.getValue();  

21.             // 这里是向table中放入一个Node的关键  

22.             putVal(hash(key), key, value, false, evict);  //***

23.         }  
```



resize()： 当capability到达threshold时进行扩容

```
1.  final Node<K,V>[] resize() {  

2.      Node<K,V>[] oldTab = table; //记录扩容前的table  

3.      int oldCap = (oldTab == null) ? 0 : oldTab.length; // 记录扩容前的容量  

4.      int oldThr = threshold; // 记录扩容前的扩容阀值  

5.      int newCap, newThr = 0; //扩容后的容量和阀值  

6.      if (oldCap > 0) {   

7.          if (oldCap >= MAXIMUM_CAPACITY) { // 如果扩容前容量已经到达上限了，没办法扩容  

8.              threshold = Integer.MAX_VALUE;  

9.              return oldTab;  

10.         }  

11.         else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY)  // 将扩容前的容量和阀值都增加一倍  

13.             newThr = oldThr << 1; // double threshold  

14.     }  

15.     else if (oldThr > 0) // table未初始化，调用的是HashMap的有参构造方法初始化了threshold  

16.         newCap = oldThr;  

17.     else {      // table未初始化 调用的HashMap的默认构造方法   

18.         newCap = DEFAULT_INITIAL_CAPACITY;  

19.         newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);  

20.     }  

21.     if (newThr == 0) { // 对应的是oldThr > 0的情况，这里根据newCap初始化newThr  

22.         float ft = (float)newCap * loadFactor;  // threshold = capability * loadFactor  

23.         newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?  (int)ft : Integer.MAX_VALUE);  

25.     }  

26.     threshold = newThr;  

27.     @SuppressWarnings({"rawtypes","unchecked"})  

28.      Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap]; // 创建新的table  

29.     table = newTab;  

30.     if (oldTab != null) { // 将oldTab数据放到newTab中，不保证newTab中Node的顺序和位置  

31.         for (int j = 0; j < oldCap; ++j) {  

32.             Node<K,V> e;  

33.             if ((e = oldTab[j]) != null) {  

34.                 oldTab[j] = null; // 清空oldTab值，方便gc  

35.                 if (e.next == null) // 扩容前的Node是光秃秃的节点时 直接放到计算索引后的位置  

36.                     newTab[e.hash & (newCap - 1)] = e; // 根据e.hash & (newCap - 1) 生成索引放到newTab中  

37.                 else if (e instanceof TreeNode) //扩容前的Node如果是红黑树节点的话 将它分离   

38.                     ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);  

39.                 else { // preserve order  

40.                     Node<K,V> loHead = null, loTail = null;  

41.                     Node<K,V> hiHead = null, hiTail = null;  

42.                     Node<K,V> next;  

43.                     // 将同一桶中的元素根据e.hash & oldCap是否为0进行分割，分成两个不同的链表，完成rehash  ，因为我们使用的是2次幂的扩展(指长度扩为原来2倍)，所以，元素的位置要么是在原位置，要么是在原位置再移动2次幂的位置

44.                     do {  

45.                         next = e.next;  

46.                         if ((e.hash & oldCap) == 0) {  

47.                             if (loTail == null)  

48.                                 loHead = e;  

49.                             else  

50.                                 loTail.next = e;  

51.                             loTail = e;  

52.                         }  

53.                         else {  

54.                             if (hiTail == null)  

55.                                 hiHead = e;  

56.                             else  

57.                                 hiTail.next = e;  

58.                             hiTail = e;  

59.                         }  

60.                     } while ((e = next) != null);  

61.                     if (loTail != null) { // (e.hash & oldCap)为0组成的链表放在当前位置  

62.                         loTail.next = null;  

63.                         newTab[j] = loHead;  

64.                     }  

65.                     if (hiTail != null) { // (e.hash & oldCap)不为0组成的链表放在j + oldCap的位置  

66.                         hiTail.next = null;  

67.                         newTab[j + oldCap] = hiHead;  

68.                     }  

69.                 }  

70.             }  

71.         }  

72.     }  

73.     return newTab;  

74. }  
```



进行扩容，会伴随着一次重新hash分配，并且会遍历hash表中所有的元素，是非常耗时的。在编写程序中，要尽量避免resize。

在resize前和resize后的元素布局如下:

![img](http://pcc.huitogo.club/9b317aabd3b474739b67a4cda0bdfb75)



putVal(hash(key), key, value, false, evict)：在table中放入一个Node

```
1.  final V putVal(int hash, K key, V value, boolean onlyIfAbsent,  // onlyIfAbsent判断在key重复时是否覆盖   

2.                   boolean evict) {    // evict参数用于LinkedHashMap中的尾部操作，这里没有实际意义      

3.        Node<K,V>[] tab; Node<K,V> p; int n, i;　//定义变量tab是将要操作的Node数组引用，p表示tab上的某Node节点，n为tab的长度，i为tab的下标。  

4.       if ((tab = table) == null || (n = tab.length) == 0)　 // table未初始化时进行初始化             　　　　  

5.           n = (tab = resize()).length;　　　　　　　　　　　　　　　　　　　　　　　　          

6.       if ((p = tab[i = (n - 1) & hash]) == null)  // 如果计算后的索引位置是否为空则直接赋值  

7.            tab[i] = newNode(hash, key, value, null);　  

8.       else {　　// 计算后索引位置有值的情况下，有三种情况：p为链表节点；p为红黑树节点；p是链表节点但长度为临界长度TREEIFY_THRESHOLD，再插入任何元素就要变成红黑树了。  

9.           Node<K,V> e; K k;　//定义e引用即将插入的Node节点， k = p.key。  

10.           if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))   // key值存在的情况下记录p，但不直接覆盖，后面需要判断onlyIfAbsent  

11.               e = p;　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　    

12.          else if (p instanceof TreeNode)  //p是红黑树节点，直接强制转型p后调用TreeNode.putTreeVal方法，返回的引用赋给e。  

13.              e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);   // putTreeVal内部会进行遍历，存在相同hash时返回被覆盖的TreeNode，否则返回null。  

14.         else {　  //  p为链表节点  

15.              for (int binCount = 0; ; ++binCount) {　　　　// 遍历p节点所在的链表

16.                  if ((e = p.next) == null) {　　　　　　　　　　　　　　　　　　　　  

17.                      p.next = newNode(hash, key, value, null); // 将当前节点添加到末尾　　　　　　　　　　  

18.                      if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st  //插入成功后，要判断是否需要转换为红黑树，因为插入后链表长度加1，而binCount并不包含新节点，所以判断时要将临界阈值减1。  

19.                         treeifyBin(tab, hash);　　//当新长度满足转换条件时，调用treeifyBin方法，将该链表转换为红黑树。 //***

20.                      break;　　//如果不满足转换条件，插入操作到此结束了，break退出  

21.                 }  

22.                   if (e.hash == hash &&　((k = e.key) == key || (key != null && key.equals(k)))) //判断当前遍历的节点的key是否相同。  

23.                      break;　//找到了相同key的节点后无需任何操作  

24.                 p = e;　　// 将e指向p 以进行下一轮遍历 因为e = p.next  

25.              }  

26.           }  

27.          if (e != null) { // 这时候开始处理 key重复的情况  

28.              V oldValue = e.value;　　　　　　　　　　　　　　　　　　　　　　　　　　  

29.              if (!onlyIfAbsent || oldValue == null)　　// 如果onlyIfAbsent 为true或者重复的key的value为空则进行覆盖　　　　　　　　　　　　　　　  
30.                   e.value = value;　　　  

31. 　　　　　afterNodeAccess(e);　　//这个函数在hashmap中没有任何操作，是个空函数，他存在主要是为了linkedHashMap的一些后续处理工作。  

32.              return oldValue;　//返回的是被覆盖的oldValue。  

33.          }  

34.      }　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　   

35.       ++modCount;　　　//值得一提的是，对key相同而覆盖oldValue的情况，在前面已经return，不会执行这里，所以那一类情况不算数据结构变化，并不改变modCount值。  

36.      if (++size > threshold)　　　//判断是否达到了扩容标准。  

37.           resize();　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　  

38.      afterNodeInsertion(evict);　　　//这里与前面的afterNodeAccess同理，是用于linkedHashMap的尾部操作，HashMap中并无实际意义。  

39.      return null;　　　 //对于真正进行插入元素的情况，put函数一律返回null。  

40.  }  
```



#### 5. HashMap的重要方法

##### 5.1 put方法

```
1.  public V put(K key, V value) { // 调用的putVal，注意onlyIfAbsent为false，即覆盖key  

2.       return putVal(hash(key), key, value, false, true);  

3.   } 
```



HashMap并没有直接提供putVal接口给用户调用，而是提供的put函数，而put函数就是通过putVal来插入元素的。



put方法的流程图如下：

![img](http://pcc.huitogo.club/ce501e471fc17d351ce46b17fbb6ebb4)



解释起来的步骤如下：

1. 检查数组是否为空，执行resize()扩充；

2. 通过hash值计算数组索引，获取该索引位的首节点。

3. 如果首节点为null，直接添加节点到该索引位。

4. 如果首节点不为null，那么有3种情况

   　A. key和首节点的key相同，覆盖value；否则执行B或C

      　B. 如果首节点是红黑树节点（TreeNode），将键值对添加到红黑树。

      　C. 如果首节点是链表，将键值对添加到链表。添加之后会判断链表长度是否到达	   TREEIFY_THRESHOLD - 1这个阈值，“尝试”将链表转换成红黑树。

最后判断当前元素个数是否大于threshold，扩充数组。



##### 5.2 get方法

**get(Object key)**

```
1.  public V get(Object key) {  

2.          Node<K,V> e;  

3.          return (e = getNode(hash(key), key)) == null ? null : e.value;  

4.      }  
```



getNode(hash(key), key)

```
1.  final Node<K,V> getNode(int hash, Object key) {  

2.      Node<K,V>[] tab; Node<K,V> first, e; int n; K k;  

3.      // table已经初始化，长度大于0，根据hash寻找table中的项也不为空  

4.      if ((tab = table) != null && (n = tab.length) > 0 &&   (first = tab[(n - 1) & hash]) != null) {  

6.          // 桶中第一项(数组元素)相等  

7.          if (first.hash == hash && // always check first node  

8.              ((k = first.key) == key || (key != null && key.equals(k))))  

9.              return first;  

10.         // 桶中不止一个结点  

11.         if ((e = first.next) != null) {  

12.             // 为红黑树结点  

13.             if (first instanceof TreeNode)  

14.                 // 在红黑树中查找  

15.                 return ((TreeNode<K,V>)first).getTreeNode(hash, key);  

16.             // 否则，在链表中查找  

17.             do {  

18.                 if (e.hash == hash &&  ((k = e.key) == key || (key != null && key.equals(k))))  

20.                     return e;  

21.             } while ((e = e.next) != null);  

22.         }  

23.     }  

24.     return null;  

25. } 
```

HashMap并没有直接提供getNode接口给用户调用，而是提供的get函数，而get函数就是通过getNode来取得元素的。



##### 5.3 remove方法

```
2.  public V remove(Object key) {  

3.      Node<K,V> e;  

4.      return (e = removeNode(hash(key), key, null, false, true)) == null ? null : e.value;  

6.  }  
```



removeNode(hash(key), key, null, false, true)：根据key移除单个节点

```
1.  final Node<K,V> removeNode(int hash, Object key, Object value,  // 这里value是在matchValue为true时进一步判断是否移除node的条件 value == node.value  

2.                             boolean matchValue, boolean movable) { // matchValue 为true时表示删除它key对应的value，不删除key；movable为false时表示删除后不移动节点  

3.      // tab 哈希数组，p 数组下标的节点，n 长度，index 当前数组下标  

4.      Node<K,V>[] tab; 
    
6.      Node<K,V> p; 

8.      int n, index;   

5.      if ((tab = table) != null && (n = tab.length) > 0 &&   (p = tab[index = (n - 1) & hash]) != null) {  

7.          //  node 存储要删除的节点，e 临时变量，k 当前节点的key，v 当前节点的value  

8.          Node<K,V> node = null, e; K k; V v;  

9.          if (p.hash == hash &&   ((k = p.key) == key || (key != null && key.equals(k)))) // 如果数组下标的节点正好是要删除的节点  

11.             node = p;  

12.         else if ((e = p.next) != null) { // 链表  

13.             if (p instanceof TreeNode) // 当前数组节点是红黑树节点  

14.                 node = ((TreeNode<K,V>)p).getTreeNode(hash, key);  

15.             else {  

16.                 do {  // 遍历链表 找到指定key的节点  

17.                     if (e.hash == hash &&  ((k = e.key) == key ||  (key != null && key.equals(k)))) {  

20.                         node = e;  

21.                         break;  

22.                     }  

23.                     p = e;  

24.                 } while ((e = e.next) != null);  

25.             }  

26.         }  

27.         if (node != null && (!matchValue || (v = node.value) == value ||  (value != null && value.equals(v)))) {  // 判断是否要完全移除节点  

29.             if (node instanceof TreeNode)  

30.                 ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);  

31.             else if (node == p) // 要移除的节点是头节点  

32.                 tab[index] = node.next;  

33.             else    

34.                 p.next = node.next; // 把要删除的下一个结点设为上一个结点的下一个节点  

35.             ++modCount;  

36.             --size;  

37.             afterNodeRemoval(node); // 此方法在hashMap中是为了让子类去实现，主要是对删除结点后的链表关系进行处理  

38.             return node;  

39.         }  

40.     }  

41.     return null;  

42. }  
```



##### 5.4 set方法

元素的修改也是put方法，因为key是唯一的，所以修改元素，就是把新值覆盖旧值。



#### 6. HashMap的迭代

 **jdk1.8底层是怎么维护entrySet()的？**

hashMap内部维护了一个entrySet集合

```
1.  /** 

2.   * Holds cached entrySet(). Note that AbstractMap fields are used 

3.   * for keySet() and values(). 

4.   */  

5.  transient Set<Map.Entry<K,V>> entrySet;
```



在hashMap遍历的时候（通过entrySet()）会从这个集合里面取值，但是在查看put()和remove()源码后发现hashMap并没有显示的维护这个entrySet集合，那么它是怎么取到值进行遍历的呢？

查看entrySet源码，发现在调用entrySet()会返回一个EntrySet对象，EntrySet有一个iterator方法返回一个EntryIterator迭代器

```
1.  final class EntrySet extends AbstractSet<Map.Entry<K,V>> {  

2.      public final int size()                 { return size; }  

3.      public final void clear()               { HashMap.this.clear(); }  

4.      public final Iterator<Map.Entry<K,V>> iterator() {  

5.          return new EntryIterator();  

6.      }  

7.      ...  

8.  } 
```



这个迭代器继承了HashIterator，那么关键在这个HashIterator

```
1.  abstract class HashIterator {  

2.      Node<K,V> next;        // next entry to return  

3.      Node<K,V> current;     // current entry  

4.      int expectedModCount;  // for fast-fail  

5.      int index;             // current slot  

6.      // ... 构造方法 重置所有变量  

7.       public final boolean hasNext() {  

8.          return next != null;  

9.      }  

10.     // 这个就是迭代时获取值的next方法  

11.     final Node<K,V> nextNode() {  

12.         Node<K,V>[] t;  

13.         Node<K,V> e = next;  

14.         if (modCount != expectedModCount)  

15.             throw new ConcurrentModificationException();  

16.         if (e == null)  

17.             throw new NoSuchElementException();  

18.         if ((next = (current = e).next) == null && (t = table) != null) {  

19.             do {} while (index < t.length && (next = t[index++]) == null);  

20.         }  

21.         return e;  

22.     }  

23.    ... 
```

现在知道问题所在了，我们在通过迭代器遍历HashMap的时候，虽然表面是从entrySet中取值，但是**底层还是从table（Node数组）中进行取值，包括判断和删除操作都是对table操作**。



#### 7. HashMap总结

1.  数组的初始容量为16，而容量是以2的次方扩充的，一是为了提高性能使用足够大的数组，二是为了能使用位运算代替取模预算。
2. 数组是否需要扩充是通过负载因子判断的，如果当前元素个数为数组容量的0.75时，就会扩充数组。这个0.75就是默认的负载因子，可由构造传入。我们也可以设置大于1的负载因子，这样数组就不会扩充，牺牲性能，节省内存。
3. 为了解决碰撞，数组中的元素是单向链表类型。当链表长度到达一个阈值时（7或8），会将链表转换成红黑树提高性能。而当链表长度缩小到另一个阈值时（6），又会将红黑树转换回单向链表提高性能，这里是一个平衡点。
4. 对于第三点补充说明，检查链表长度转换成红黑树之前，还会先检测当前数组数组是否到达一个阈值（64），如果没有到达这个容量，会放弃转换，先去扩充数组。所以上面也说了链表长度的阈值是7或8，因为会有一次放弃转换的操作。