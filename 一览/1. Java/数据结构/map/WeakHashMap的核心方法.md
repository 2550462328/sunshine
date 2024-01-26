#### 1. weakHashMap的特性

最重要的总结当然是放在前面啦~~~~

1. weakHashMap是允许key为空
2. weakHashMap是一个会自动清除Entry的Map
3. weakHashMap的操作与HashMap完全一致
4. WeakHashMap内部数据结构是数组+链表
5. WeakHashMap常被用作缓存

WeakHashMap 特殊之处在于 WeakHashMap 里的**entry可能会被垃圾回收器自动删除**，也就是说即使你没有调用remove()或者clear()方法，它的entry也可能会慢慢变少。所以多次调用比如isEmpty，containsKey，size等方法时可能会返回不同的结果。



**Q1：为什么WeakHashMap的entry会被回收？**

在HashMap中我们都知道entry并不会无缘无故的丢失，除非进行了remove操作。

在weakHashMap中，关键在于**弱引用**，WeakHashMap中的key是间接保存在弱引用中的，所以当key没有被继续使用时，就可能会在GC的时候被回收掉。

只有**key对象是使用弱引用保存**的，**value对象实际上仍旧是通过普通的强引用来保持的**，所以应该确保value不会直接或者间接的保持其对应key的强引用，因为这样会阻止key被回收。



#### 2. weakHashMap的结构

WeakHashMap并不是继承自HashMap，而是继承自AbstractMap，跟HashMap的继承结构差不多

![img](http://pcc.huitogo.club/48a0005ebbf866a7f3223e36031052b2)



WeakHashMap中的数据结构是数组+链表的形式，这一点跟HashMap也是一致的，但不同的是，在JDK8中，当发生较多key冲突的时候，HashMap中会由链表转为红黑树，而WeakHashMap则一直使用链表进行存储。

![img](http://pcc.huitogo.club/a8601f03b839d8e92cac14012b78be25)



#### 3. 成员变量

```
3.  private static final int DEFAULT_INITIAL_CAPACITY = 16;  

4.  // 最大容量  

5.  private static final int MAXIMUM_CAPACITY = 1 << 30;  

6.  // 默认装载因子  

7.  private static final float DEFAULT_LOAD_FACTOR = 0.75f;  

8.  // Entry数组，长度必须为2的幂  

9.  Entry<K,V>[] table;  

10. // 元素个数  

11. private int size;  

12. // 阈值   

13. private int threshold;  

14. // 装载因子  

15. private final float loadFactor;  

16. // 引用队列  

17. private final ReferenceQueue<Object> queue = new ReferenceQueue<>();  

18. // 修改次数  

19. int modCount; 
```

这里主要是这个ReferenceQueue，用来存放那些已经被回收了的弱引用对象。



#### 4. 构造方法

##### 4.1 无参

```
3.  public WeakHashMap() {  

4.      this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR);  

5.  }  
```



##### 4.2 带初始容量

```
7.  public WeakHashMap(int initialCapacity) {  

8.      this(initialCapacity, DEFAULT_LOAD_FACTOR);  

9.  } 
```



##### 4.3 带初始容量和装载因子

```
11. public WeakHashMap(int initialCapacity, float loadFactor) {  

12.     // 校验initialCapacity  

13.     if (initialCapacity < 0)  

14.         throw new IllegalArgumentException("Illegal Initial Capacity: "+ initialCapacity);  

16.     if (initialCapacity > MAXIMUM_CAPACITY)  

17.         initialCapacity = MAXIMUM_CAPACITY;  

18.     // 校验loadFactor  

19.     if (loadFactor <= 0 || Float.isNaN(loadFactor))  

20.         throw new IllegalArgumentException("Illegal Load factor: "+ loadFactor);  

22.     int capacity = 1;  

23.     // 将容量设置为大于initialCapacity的最小2的幂  

24.     while (capacity < initialCapacity)  

25.         capacity <<= 1;  

26.     table = newTable(capacity);  

27.     this.loadFactor = loadFactor;  

28.     threshold = (int)(capacity * loadFactor);  

29. }  
```



##### 4.4 带集合

```
31. public WeakHashMap(Map<? extends K, ? extends V> m) {  

32.     this(Math.max((int) (m.size() / DEFAULT_LOAD_FACTOR) + 1, DEFAULT_INITIAL_CAPACITY), DEFAULT_LOAD_FACTOR);  

35.     putAll(m);  

36. } 
```



#### 5. Entry源码

上面我们知道weakHashMap的核心就是entry

Entry继承自WeakReference，继承关系图如下：

![img](http://pcc.huitogo.club/1bba1422eb4e9da669cc1f80cc844ff5)



再来看下entry源码

```
1.  private static class Entry<K,V> extends WeakReference<Object> implements Map.Entry<K,V> {  

2.      V value;  

3.      final int hash;  

4.      Entry<K,V> next;  

5.      /** 

6.       * Creates new entry. 

7.       */  

8.      Entry(Object key, V value, ReferenceQueue<Object> queue,  int hash, Entry<K,V> next) {  

11.         super(key, queue); 

12.         this.value = value;  

13.         this.hash  = hash;  

14.         this.next  = next;  

15.     }  

16. }  
```

在entry的构造方法中我们可以发现对于形参key，被父类构造方法调用，在weakRefrence中指定refrent和queue。



在entry中需要主要的方法是getKey()方法，获取entry的key值。

```
1.  public K getKey() {  

2.      return (K) WeakHashMap.unmaskNull(get());  

3.  }  
```

这个方法是获取之前构造方法中传入weakRefence中的refrent值，值得注意的是这里如果从weakRefrence中取到的regfrent为null的话会返回一个特殊变量，而不是直接返回null

```
1. private static final Object NULL_KEY = new Object();  
```



所以需要进行转换，当然也有对应的反转换，将null转换成new Object()的

```
1.  static Object unmaskNull(Object key) {  

2.      return (key == NULL_KEY) ? null : key;  

3.  }  


4.   private static Object maskNull(Object key) {  

5.      return (key == null) ? NULL_KEY : key;  

6.  }  
```



可以看出在构建entry的时候就已经把key交给了weakRefrence保管，然后在遍历取值等操作的时候都会做一次清除操作，也就是消除key为null的entry。但是在weakHashMap不是直接遍历key为null的entry，而是**从refrenceQueue中取值**，前面我们知道**refrenceQueue存放的就是被回收的key**，所以这样效率更高一些，对应核心代码如下：

```
1.  private void expungeStaleEntries() {  

2.      for (Object x; (x = queue.poll()) != null; ) {  

3.  // 循环获取引用队列中的对象

4.          synchronized (queue) {  

5.              @SuppressWarnings("unchecked")  

6.               Entry<K,V> e = (Entry<K,V>) x;  

7.              int i = indexFor(e.hash, table.length);  

8.  // 找到之前的Entry

9.              Entry<K,V> prev = table[i];  

10.             Entry<K,V> p = prev;  

11. // 在链表中寻找

12.             while (p != null) {  

13.                 Entry<K,V> next = p.next;  

14.                 if (p == e) {  

15.                     if (prev == e)  

16.                         table[i] = next;  

17.                     else  

18.                         prev.next = next;  

19.                     // 将对应的value置为null，帮助GC回收 

20.                     e.value = null; // Help GC  

21.                     size--;  

22.                     break;  

23.                 }  

24.                 prev = p;  

25.                 p = next;  

26.             }  

27.         }  

28.     }  

29. }  
```



#### 6. weakHashMap的主要方法

这里我们主要感受以下weakHashMap在为key取hashCode的算法以及扩容策略跟hashMap有什么不同



##### 6.1 put

```
2.  public V put(K key, V value) {  

3.      // 处理null值  

4.      Object k = maskNull(key);  

5.      // 计算hash  

6.      int h = hash(k);  

7.      // 获取table  

8.      Entry<K,V>[] tab = getTable();  

9.      // 计算下标  

10.     int i = indexFor(h, tab.length);  

11.     // 查找Entry  

12.     for (Entry<K,V> e = tab[i]; e != null; e = e.next) {  

13.         if (h == e.hash && eq(k, e.get())) {  

14.             V oldValue = e.value;  

15.             if (value != oldValue)  

16.                 e.value = value;  

17.             return oldValue;  

18.         }  

19.     }  

20.     modCount++;  

21.     Entry<K,V> e = tab[i];  

22.     tab[i] = new Entry<>(k, value, queue, h, e);  

23.     // 如果元素个数超过阈值，则进行扩容  

24.     if (++size >= threshold)  

25.         resize(tab.length * 2);  

26.     return null;  

27. }  
```



这个方法里面已经有我们要的答案了。

**hash(k)**：根据key计算hashCode

```
1.  final int hash(Object k) {  

2.      int h = k.hashCode();  

3.      // 这里做了二次散列，来扩大低位的影响  

4.      h ^= (h >>> 20) ^ (h >>> 12);  

5.      return h ^ (h >>> 7) ^ (h >>> 4);  

6.  }  
```

在hashMap中计算hashCode直接将key的hashCode的高16位和低16位做了一次异或（散列）操作。

然而在weakHashMap中显得复杂许多，在hash中作了两次散列操作，主要是为了扩大低位的影响，**因为Entry数组的大小是2的幂，在进行查找的时候，进行掩码处理，如果不进行二次散列，那么低位对index就完全没有影响了。**



**indexFor(h, tab.length)**：计算下标

```
1.  private static int indexFor(int h, int length) {  

2.      return h & (length-1);  

3.  }  
```

这个和HashMap操作一样。



**getTable()**：获取最新的table

```
1.  private Entry<K,V>[] getTable() {  

2.      // 清除被回收的Entry对象  

3.      expungeStaleEntries();  

4.      return table;  

5.  }  
```

这里就可以看到进行了清除操作，清除被回收的对象。



**resize(tab.length \* 2)**：扩容操作

```
1.  void resize(int newCapacity) {  

2.      // 获取当前table  

3.      Entry<K,V>[] oldTable = getTable();  

4.      int oldCapacity = oldTable.length;  

5.      if (oldCapacity == MAXIMUM_CAPACITY) {  

6.          threshold = Integer.MAX_VALUE;  

7.          return;  

8.      }  

9.      // 新建一个table  

10.     Entry<K,V>[] newTable = **newTable**(newCapacity);  

11.     // 将旧table中的内容复制到新table中  

12.     transfer(oldTable, newTable);  

13.     table = newTable;  

14.     if (size >= threshold / 2) {  

15.         threshold = (int)(newCapacity * loadFactor);  

16.     } else {  

17.         expungeStaleEntries();  

18.         transfer(newTable, oldTable);  

19.         table = oldTable;  

20.     }  

21. }  


22. // 新建Entry数组  

23. private Entry<K,V>[] newTable(int n) {  

24.     return (Entry<K,V>[]) new Entry<?,?>[n];  

25. }  


26. private void transfer(Entry<K,V>[] src, Entry<K,V>[] dest) {  

27.     for (int j = 0; j < src.length; ++j) {  

28.         Entry<K,V> e = src[j];  

29.         src[j] = null;  

30.         while (e != null) {  

31.             Entry<K,V> next = e.next;  

32.             Object key = e.get();  

33.             if (key == null) {  

34.                 e.next = null;   

35.                 e.value = null;   

36.                 size--;  

37.             } else {  

38.                 int i = indexFor(e.hash, dest.length);  

39.                 e.next = dest[i];  

40.                 dest[i] = e;  

41.             }  

42.             e = next;  

43.         }  

44.     }  

45. }  
```

这里和HashMap一样，也是扩充2的幂次方，但是有个有意思的地方就是，这里为了防止在扩容过程中，entry又进行了一轮回收导致释放了大量空间，所以在扩容后又进行了一次判断，如果size < threshold / 2的话就进行还原操作。



##### 6.2 get

```
2.  public V get(Object key) {  

3.      // 对null值特殊处理  

4.      Object k = maskNull(key);  

5.      // 取key的hash值  

6.      int h = hash(k);  

7.      // 取当前table  

8.      Entry<K,V>[] tab = getTable();  

9.      // 获取下标  

10.     int index = indexFor(h, tab.length);  

11.     Entry<K,V> e = tab[index];  

12.     // 链表中查找元素  

13.     while (e != null) {  

14.         if (e.hash == h && **eq(k, e.get()**))  

15.             return e.value;  

16.         e = e.next;  

17.     }  

18.     return null;  

19. }  
```



**eq(k, e.get())**：对比key值

```
1.  private static boolean eq(Object x, Object y) {  

2.      return x == y || x.equals(y);  

3.  }  
```



##### 6.3 remove

```
5.  public V remove(Object key) {  

6.      // 对null值特殊处理  

7.      Object k = maskNull(key);  

8.      // 取key的hash  

9.      int h = hash(k);  

10.     // 取当前table  

11.     Entry<K,V>[] tab = getTable();  

12.     // 计算下标  

13.     int i = indexFor(h, tab.length);  

14.     Entry<K,V> prev = tab[i];  

15.     Entry<K,V> e = prev;  

16.     while (e != null) {  

17.         Entry<K,V> next = e.next;  

18.         // 查找对应Entry  

19.         if (h == e.hash && eq(k, e.get())) {  

20.             modCount++;  

21.             size--;  

22.             if (prev == e)  

23.                 tab[i] = next;  

24.             else  

25.                 prev.next = next;  

26.             // 如果找到，返回对应Entry的value  

27.             return e.value;  

28.         }  

29.         prev = e;  

30.         e = next;  

31.     }  

32.     return null;  

33. }  
```



#### 7. weakHashMap的应用场景

##### 7.1 用作缓存

示例如下：

```
1.  public class MyWeakHashMap<K, V> {  

2.      private int size;  

3.      private ConcurrentHashMap<K, V> useMap;  

4.      private WeakHashMap<K, V> cacheMap;  


5.      public MyWeakHashMap(int size, ConcurrentHashMap<K, V> useMap, WeakHashMap<K, V> cacheMap) {  

6.          super();  

7.          this.size = size;  

8.          this.useMap = useMap;  

9.          this.cacheMap = cacheMap;  

10.     }  


11.     public void put(K k, V v) {  

12.         if (useMap.size() >= size) {  

13.             synchronized (cacheMap) {  

14.                 **cacheMap.putAll(useMap);**  

15.                 **cacheMap.clear(); ** 

16.             }  

17.         }  

18.         useMap.put(k, v);  

19.     }  


20.     public V get(K k) {  

21.         V v = useMap.get(k);  

22.         if (v == null) {  

23.             synchronized (cacheMap) {  

24.                 v = cacheMap.get(k);  

25.             }  

26.             if (v != null) {  

27.                 **useMap.put(k, cacheMap.get(k));**  

28.             }  

29.         }  

30.         return v;  

31.     }  

32. }  
```

这样设计的好处是，能将相对常用的对象都能在useMap中找到，**不常用的则存入cacheMap中缓存**，并且由于WeakHashMap能自动清除Entry，所以**不用担心cacheMap中键值对过多而导致OOM**。



##### 7.2 配合事务

配合事务进行使用，存储事务过程中的各类信息，可以使用如下结构：

```
1. WeakHashMap<String,Map<K,V>> transactionCache; 
```

这里key为String类型，可以用来标志区分不同的事务，起到一个事务id的作用。value是一个map，可以是一个简单的HashMap或者LinkedHashMap，用来存放在事务中需要使用到的信息。

在事务开始时创建一个事务id，并用它来作为key，事务结束后，将这个强引用消除掉，这样既能保证在事务中可以获取到所需要的信息，又能自动释放掉map中的所有信息。



#### 8. WeakHashMap的注意事项

**1）如果key是一个常量，即使将key置为null，对应的entry也不会被回收**



举例说明，下面gc后不会清除已经为null的s1。

```
1.  WeakHashMap<Object, String> map = new WeakHashMap<>();  

2.  Integer s1 = 118;  

3.  String s2 = "s2";  

4.  map.put(s1, "s1");  

5.  map.put(s2, "s2");  

6.  System.out.println("gc前，map=" + map);  

7.  s1 = null;  

8.  System.out.println(s1);  

9.  System.gc();  

10. System.out.println("gc后，map=" + map); 
```