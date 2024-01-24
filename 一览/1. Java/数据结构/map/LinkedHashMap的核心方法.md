### 1. LinkedHashMap和HashMap

linkedHashMap的继承关系如下图：

![img](http://pcc.huitogo.club/04c55ea76194a39cf961f26a61b04832)



可以看出linkedHashMap是HashMap的子类，拥有HashMap的所有特性，比如高效的查找元素，同时，**LinkedHashMap还保持着内部元素的顺序**，可以根据**元素访问时间**或**元素插入时间**进行排序，**因此适用于需保持元素顺序的场景**。另外，由于LinkedHashMap有保持元素访问顺序的模式，所以也常被用作LRU缓存（Least-Recently-Used Cache，即最近最少使用缓存策略）



**Q1：LinkedHashMap是怎么实现元素遍历的有序性的？**

通过源码可以看到组成LinkedHashMap的元素节点，如下：

```
1.  static class Entry<K,V> extends HashMap.Node<K,V> {  

2.      Entry<K,V> before, after;  

3.      Entry(int hash, K key, V value, Node<K,V> next) {  

4.          super(hash, key, value, next);  

5.      }  

6.  }  
```

可以看到LinkedHashMap中的元素除了维护HashMap的结构（数组+链表）外，还维护了一个双链表，正是因为**这个双链表才能让其中的元素保持某种顺序**。

由于所有元素使用链表相连，所以遍历的效率略高于HashMap，因为HashMap遍历时，需要每个桶中先遍历到链表尾部，然后再遍历到下一个桶，当元素不多而空桶数量很多时，就会有很多次的无效访问，所以理论上来说，**LinkedHashMap的遍历效率是要比HashMap高的**。



根据下图HashMap和LinkedHashMap的结构图可以看出：

![img](http://pcc.huitogo.club/b169adf1efd70134f9777157ebb9236e)



![img](http://pcc.huitogo.club/3eb96935e4e87afcbee2e5743664f552)



**Q2：什么HashMap中的TreeNode先继承LinkedHashMap中的Entry，而不是直接继承HashMap的Node？**

如果TreeNode直接继承自Node，则失去了链表的特性，LinkedHashMap继承HashMap后，则**需要再使用另一个内部类来继承TreeNode来使得其具有链表特性**，并且相关方法均需要重写，**大大的增加了工作量**，并且**代码的可维护性会下降**很多，另外，HashMap只会在桶中元素达到一定数量的时候才会将节点转换TreeNode，在哈希表散列良好的情况下，**TreeNode是很少被使用**到的，所以这一点点的空间浪费是值得的。



### 2. LinkedHashMap的插入和删除

在之前我们讨论一个集合进行增删的时候原理就是维护底层的数据结构，这里LinkedHashMap也会理所当然的想到维护那个Entry的双链表。

那既然不是直接维护双链表的话，那底层是怎么维护的？



现在回顾一下上节HashMap源码中的put方法中主要方法putVal()

```
1.  final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {  

3.      Node<K,V>[] tab; Node<K,V> p; int n, i;  

4.      if ((tab = table) == null || (n = tab.length) == 0)  

5.          n = (tab = resize()).length;  

6.      if ((p = tab[i = (n - 1) & hash]) == null)  

7.          tab[i] = newNode(hash, key, value, null);  //newNode方法在LinkedHashMap中被覆盖  

8.      else {  

9.          Node<K,V> e; K k;  

10.         if (p.hash == hash &&  ((k = p.key) == key || (key != null && key.equals(k))))  

12.             e = p;  

13.         else if (p instanceof TreeNode)  

14.             e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);  

15.         else {  

16.             for (int binCount = 0; ; ++binCount) {  

17.                 if ((e = p.next) == null) {  

18.                     p.next = newNode(hash, key, value, null);  

19.                     if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st  

20.                         treeifyBin(tab, hash);  

21.                     break;  

22.                 }  

23.                 if (e.hash == hash &&   ((k = e.key) == key || (key != null && key.equals(k))))  

25.                     break;  

26.                 p = e;  

27.             }  

28.         }  

29.         if (e != null) { // existing mapping for key  

30.             V oldValue = e.value;  

31.             if (!onlyIfAbsent || oldValue == null)  

32.                 e.value = value;  

33.             afterNodeAccess(e); //关键方法A  

34.             return oldValue;  

35.         }  

36.     }  

37.     ++modCount;  

38.     if (++size > threshold)  

39.         resize();  

40.     afterNodeInsertion(evict); //关键方法B  

41.     return null;  

42. } 
```



在上述源码中，先看 afterNodeAccess(e)和afterNodeInsertion(true)以及在remove()方法中对应的afterNodeRemoval(e)，LinkedHashMap的核心也就是这些操作。



**afterNodeAccess(e)**：在节点被访问之后进行双链表的调整（仅仅在accessOrder为true时进行，把当前访问的元素移动到链表尾部，这里的accessOrder为true时表示当前双链表按照访问时间排序，为false时也就是默认是插入时间排序）

```
1.  void afterNodeAccess(Node<K,V> e) { // move node to last  

2.       LinkedHashMap.Entry<K,V> last;  

3.       if (accessOrder && (last = tail) != e) {  

4.           LinkedHashMap.Entry<K,V> p =    (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;  

6.           p.after = null;  

7.           if (b == null)  

8.               head = a;  

9.           else  

10.              b.after = a;  

11.          if (a != null)  

12.              a.before = b;  

13.          else  

14.              last = b;  

15.          if (last == null)  

16.              head = p;  

17.          else {  

18.              p.before = last;  

19.              last.after = p;  

20.          }  

21.          tail = p;  

22.          ++modCount;  

23.      }  

24.  } 
```



**afterNodeInsertion(true)**：在节点被插入后进行双链表的调整，插入节点时会判断是否需要移除链表头节点，默认实现是不移除的，可以通过继承LinkedHashMap并覆盖removeEldestEntry方法来改变该特性

```
1.  void afterNodeInsertion(boolean evict) { // possibly remove eldest  

2.      LinkedHashMap.Entry<K,V> first;  

3.      // 根据条件判断是否移除最近最少被访问的节点  

4.      if (evict && (first = head) != null && removeEldestEntry(first)) {  

5.          K key = first.key;  

6.          removeNode(hash(key), key, null, false, true);  

7.      }  

8.  }  


9.  // 移除最近最少被访问条件之一，通过覆盖此方法可实现不同策略的缓存  

10. protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {  

11.     return false;  

12. } 
```

我们可以继承LinkedHashMap类，**覆盖removeEldestEntry方法实现LRU缓存策略**。



**afterNodeRemoval(e)**：在节点被移除后进行双链表的调整，移除调整其实很简单，就是把要移除的节点前一个节点的after引用指向后一个节点，把后一个节点的before引用指向前一个节点

```
1.  void afterNodeRemoval(Node<K,V> e) {  

2.      LinkedHashMap.Entry<K,V> p =   (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;  

4.      p.before = p.after = null;  

5.      if (b == null)  

6.          head = a;  

7.      else  

8.          b.after = a;  

9.      if (a == null)  

10.         tail = b;  

11.     else  

12.         a.before = b;  

13. }
```



除了这样在增删时候对节点的位移操作外，还有一个LinkedHashMap覆盖父类的newNode()方法，这个方法就是新建一个节点，插入到现有双链表的尾部

```
1.  Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {  

2.      LinkedHashMap.Entry<K,V> p =   new LinkedHashMap.Entry<K,V>(hash, key, value, e);  

4.      //新建节点，然后链接到双链表的尾部  

5.      linkNodeLast(p);  

6.      return p;  

7.  }  


8.  private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {  

9.      LinkedHashMap.Entry<K,V> last = tail;  

10.     tail = p;  

11.     if (last == null)  

12.         head = p;  

13.     else {  

14.         p.before = last;  

15.         last.after = p;  

16.     }  

17. }  
```



总结一下在LinkedHashMap中维护双链表的过程，插入节点的时候，会调用覆盖过的newNode方法，将插入的元素链接的链表尾部，删除节点的时候，将该节点的前后节点相连即可，当节点被访问时，可以将其放到链表尾部



### 3. LinkedHashMap的内部类

内部类除了Entry外还有迭代器类，键、值以及键值对的集合类

#### 3.1 LinkedHashIterator类

```
1.  abstract class LinkedHashIterator {  

2.      LinkedHashMap.Entry<K,V> next;  

3.      LinkedHashMap.Entry<K,V> current;  

4.      int expectedModCount;  


5.      LinkedHashIterator() {  

6.          next = head;  

7.          expectedModCount = modCount;  

8.          current = null;  

9.      }  


10.     public final boolean hasNext() {  

11.         ...  

12.     }  

13.     final LinkedHashMap.Entry<K,V> nextNode() {  

14.         ...  

15.     }  

16.     public final void remove() {  

17.         ...  

18.     }  

19. }  
```



这是一个抽象类，实现了迭代器的主要方法，hashNext，remove，nextNode方法，相当于一个类模板，其他迭代器类只需要继承该类然后添加一个next方法即可

```
1.  final class LinkedKeyIterator extends LinkedHashIterator  implements Iterator<K> {  

3.      public final K next() { return nextNode().getKey(); }  

4.  }  

5.  final class LinkedValueIterator extends LinkedHashIterator implements Iterator<V> {  

7.      public final V next() { return nextNode().value; }  

8.  }  

9.  final class LinkedEntryIterator extends LinkedHashIterator implements Iterator<Map.Entry<K,V>> {  

11.     public final Map.Entry<K,V> next() { return nextNode(); }  

12. } 
```



#### 3.2  键、值以及键值对的集合类

三个集合视图类结构基本一致，跟HashMap中的对应集合视图其实基本是一毛一样的，只是在迭代器方法中返回各自的内部迭代器实例而已。其余逻辑基本一致。



### 4. LinkedHashMap总结

在日常开发中，LinkedHashMap 的使用频率虽不及HashMap，但它也是不可或缺的重要实现。在 Java集合框架中，**HashMap**、**LinkedHashMap** 和 **TreeMap**三个映射类基于不同的数据结构，并实现了不同的功能。

HashMap底层基于拉链式的散列结构，并在 JDK 1.8中引入红黑树优化过长链表的问题。基于这样结构，HashMap可提供高效的增删改查操作。

LinkedHashMap在其之上，通过维护一条双向链表，实现了散列数据结构的有序遍历。TreeMap底层基于红黑树实现，利用红黑树的性质，实现了键值对排序功能。