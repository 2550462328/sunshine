#### 1. PriorityQueue的数据结构

- **PriorityQueue的内部结构是按照小顶堆的结构进行存储的**
- **PriorityQueue不允许有空元素**

PriorityQueue也是Queue的一个继承者，相比于一般的列表，它的特点便如它的名字一样，出队的时候可以按照优先级进行出队，所以不像LinkedList那样只能按照插入的顺序出队，PriorityQueue是可以根据给定的优先级顺序进行出队的。这里说的给定优先级顺序既可以是内部比较器，也可以是外部比较器。PriorityQueue内部是根据小顶堆的结构进行存储的，所谓小顶堆的意思，便是最小的元素总是在最上面，每次出队总是将堆顶元素移除，这样便能让出队变得有序。



**Q1：什么是堆？**

堆（英语：Heap）是计算机科学中一类特殊的数据结构的统称。堆通常是一个可以被看做一棵树的数组对象。在队列中，调度程序反复提取队列中第一个作业并运行，因为实际情况中某些时间较短的任务将等待很长时间才能结束，或者某些不短小，但具有重要性的作业，同样应当具有优先权。堆即为解决此类问题设计的一种数据结构。



**Q2：什么是二叉堆？**

二叉堆（英语：binary heap）是一种特殊的堆，**二叉堆是完全二叉树或者是近似完全二叉树**。二叉堆满足堆特性：父节点的键值总是保持固定的序关系于任何一个子节点的键值，且每个节点的左子树和右子树都是一个二叉堆。



**Q3：什么是完全二叉树？**

在满二叉树的基础上叶子节点可以不是满的。



**Q4：什么是大顶堆和小顶堆？**

当父节点的键值总是大于或等于任何一个子节点的键值时为大顶堆。

![img](http://pcc.huitogo.club/1067f009d7b7574ef662c8927ddd43b1)



当父节点的键值总是小于或等于任何一个子节点的键值时为小顶堆。

![img](http://pcc.huitogo.club/16bc42d3bca51e23d90cf31054fb520a)



大顶堆或者小顶堆因为自己的特点，所以可以用数组表示

下面是一个由10，16，20，22，18，25，26，30，24，23构成的小顶堆：

![img](http://pcc.huitogo.club/d60d322460e29b0e0b1482fdb23b2628)

将其从第一个元素开始依次从上到下，从左到右给每个元素添加一个序号，从0开始，这样就得到了相应元素在数组中的位置，而且这个序号是很有规则的，第**k个元素的左孩子的序号为2k+1，右孩子的序号为2k+2**，这样就很容易根据序号直接算出对应孩子的位置，**时间复杂度为o(1)**。这也就是为什么可以用数组来存储堆结构的原因了



#### 2. PriorityQueue的继承结构

![img](http://pcc.huitogo.club/727c7543dfb2befe35d8114b8d516709)



#### 3. 内部成员

```
3.  // 默认初始化容量  

4.  private static final int DEFAULT_INITIAL_CAPACITY = 11;  

5.  /** 

6.   * 优先级队列是使用平衡二叉堆表示的: 节点queue[n]的两个孩子分别为 

7.   * queue[2*n+1] 和 queue[2*(n+1)].  队列的优先级是由比较器或者 

8.   * 元素的自然排序决定的， 对于堆中的任意元素n，其后代d满足：n<=d 

9.   * 如果堆是非空的，则堆中最小值为queue[0]。 

10.  */  

11. transient Object[] queue;   

12. /** 

13.  * 队列中元素个数 

14.  */  

15. private int size = 0;  

16. /** 

17.  * 比较器 

18.  */  

19. private final Comparator<? super E> comparator;  

20. /** 

21.  * 修改次数 

22.  */  

23. transient int modCount = 0;   
```

内部使用的是一个Object数组进行元素的存储，并对该数组进行了详细的注释，所以不管是根据子节点找父节点，还是根据父节点找子节点都非常的方便。



#### 4. 构造函数

1）使用指定容量创建一个优先级队列，并使用指定比较器进行排序

```
3.  /** 

4.   * 使用指定容量创建一个优先级队列，并使用指定比较器进行排序。 

5.   * 但如果指定的容量小于1则会抛出异常 

6.   */  

7.  public PriorityQueue(int initialCapacity,  Comparator<? super E> comparator) {  

9.      if (initialCapacity < 1)  

10.         throw new IllegalArgumentException();  

11.     this.queue = new Object[initialCapacity];  

12.     this.comparator = comparator;  

13. }  
```



2）使用指定集合的所有元素构造一个优先级队列

```
15. /** 

16.  * 使用指定集合的所有元素构造一个优先级队列， 

17.  * 如果该集合为SortedSet或者PriorityQueue类型，则会使用相同的顺序进行排序， 

18.  * 否则，将使用元素的自然排序（此时元素必须实现comparable接口），否则会抛出异常 

19.  * 并且集合中不能有null元素，否则会抛出异常 

20.  */  

21. @SuppressWarnings("unchecked")  

22. public PriorityQueue(Collection<? extends E> c) {  

23.     if (c instanceof SortedSet<?>) {  

24.         SortedSet<? extends E> ss = (SortedSet<? extends E>) c;  

25.         this.comparator = (Comparator<? super E>) ss.comparator();  

26.         **initElementsFromCollection(ss)**;  

27.     }  

28.     else if (c instanceof PriorityQueue<?>) {  

29.         PriorityQueue<? extends E> pq = (PriorityQueue<? extends E>) c;  

30.         this.comparator = (Comparator<? super E>) pq.comparator();  

31.         **initFromPriorityQueue(pq)**;  

32.     }  

33.     else {  

34.         this.comparator = null;  

35.         **initFromCollection(c)**;  

36.     }  

37. }  
```



从集合中构造优先级队列的时候，调用了几个初始化函数

initFromPriorityQueue(pq)代码

```
1.  private void initFromPriorityQueue(PriorityQueue<? extends E> c) {  

2.      if (c.getClass() == PriorityQueue.class) {  

3.          this.queue = c.toArray();  

4.          this.size = c.size();  

5.      } else {  

6.          initFromCollection(c);  

7.      }  

8.  }  
```



initFromCollection(c)代码

```
1.  private void initFromCollection(Collection<? extends E> c) {  

2.      initElementsFromCollection(c);  

3.      heapify();  

4.  }  
```



initElementsFromCollection(ss)代码

```
1.  private void initElementsFromCollection(Collection<? extends E> c) {  

2.      Object[] a = c.toArray();  

3.      // If c.toArray incorrectly doesn't return Object[], copy it.  

4.      if (a.getClass() != Object[].class)  

5.          a = Arrays.copyOf(a, a.length, Object[].class);  

6.      int len = a.length;  

7.      if (len == 1 || this.comparator != null)  

8.          for (int i = 0; i < len; i++)  

9.              if (a[i] == null)  

10.                 throw new NullPointerException();  

11.     this.queue = a;  

12.     this.size = a.length;  

13. }  
```



initFromPriorityQueue从另外一个优先级队列构造一个新的优先级队列，此时内部的数组元素不需要进行调整，只需要将原数组元素都复制过来即可。但是从其他非PriorityQueue的集合中构造优先级队列时，需要先将元素复制过来后再进行调整，调整主要使用的是上面的heapify方法：

```
1.  private void heapify() {  

2.      // 从最后一个非叶子节点开始从下往上调整  

3.      for (int i = (size >>> 1) - 1; i >= 0; i--)  

4.         siftDown(i, (E) queue[i]);  

5.  }  


6.  // 划重点了，这个函数即对应上面的元素删除时从上往下调整的步骤  

7.  private void siftDown(int k, E x) {  

8.      if (comparator != null)  

9.          // 如果比较器不为null，则使用比较器进行比较  

10.         siftDownUsingComparator(k, x);  

11.     else  

12.         // 否则使用元素的compareTo方法进行比较  

13.         siftDownComparable(k, x);  

14. }  


15. private void siftDownUsingComparator(int k, E x) {  

16.     // 使用half记录队列size的一半，如果比half小的话，说明不是叶子节点  

17.     // 因为最后一个节点的序号为size - 1，其父节点的序号为(size - 2) / 2或者(size - 3 ) / 2  

18.     // 所以half所在位置刚好是第一个叶子节点  

19.     int half = size >>> 1;  

20.     while (k < half) {  

21.         // 如果不是叶子节点，找出其孩子中较小的那个并用其替换  

22.         int child = (k << 1) + 1;  

23.         Object c = queue[child];  

24.         int right = child + 1;  

25.         if (right < size && comparator.compare((E) c, (E) queue[right]) > 0)  

27.             c = queue[child = right];  

28.         if (comparator.compare(x, (E) c) <= 0)  

29.             break;  

30.         // 用c替换  

31.         queue[k] = c;  

32.         k = child;  

33.     }  

35.     queue[k] = x;  

36. }  

37. // 同上，只是比较的时候使用的是元素的compareTo方法  


38. private void siftDownComparable(int k, E x) {  

39.     Comparable<? super E> key = (Comparable<? super E>)x;  

40.     int half = size >>> 1;        // 如果是非叶子节点则继续循环  

41.     while (k < half) {  

42.         int child = (k << 1) + 1;  

43.         Object c = queue[child];  

44.         int right = child + 1;  

45.         if (right < size && ((Comparable<? super E>) c).compareTo((E) queue[right]) > 0)  

47.             c = queue[child = right];  

48.         if (key.compareTo((E) c) <= 0)  

49.             break;  

50.         queue[k] = c;  

51.         k = child;  

52.     }  

53.     queue[k] = key;  

54. }  
```



上述调整的过程来举个例说明一下：

比如初始集合是{14,7,12,6,9,4,17,23,10,15,3}，调整过程如下：

**从最后一个非叶子节点从下往上调整**

![img](http://pcc.huitogo.club/3a8db0c7bce1eaa992d2c180862ece78)

![img](http://pcc.huitogo.club/937bd7bde0c028083f00c602c1b2ea33)



siftDown( 3,queue[3])不需要调整，跳过

![img](http://pcc.huitogo.club/e3a0857b9d1c9f533c189c75ada82482)



siftDown( 1,queue[1])不需要调整，跳过

![img](http://pcc.huitogo.club/109fb4dc013391077d1ba326c825d1c2)



这样最小的元素就被顶上去了，有没有觉得有点像冒泡排序

有对应的向下调整自然有相应的向上调正，并且**siftDown和siftUp是PriorityQueue的核心方法**。下面是siftUp的代码，和siftDown的实现十分类似。

```
1.  private void siftUp(int k, E x) {  

2.      if (comparator != null)  

3.          siftUpUsingComparator(k, x);  

4.      else  

5.          siftUpComparable(k, x);  

6.  }  


7.  @SuppressWarnings("unchecked")  

8.  private void siftUpComparable(int k, E x) {  

9.      Comparable<? super E> key = (Comparable<? super E>) x;  

10.     while (k > 0) {  

11.         int parent = (k - 1) >>> 1;  

12.         Object e = queue[parent];  

13.         if (key.compareTo((E) e) >= 0)  

14.             break;  

15.         queue[k] = e;  

16.         k = parent;  

17.     }  

18.     queue[k] = key;  

19. }  


20. @SuppressWarnings("unchecked")  

21. private void siftUpUsingComparator(int k, E x) {  

22.     while (k > 0) {  

23.         int parent = (k - 1) >>> 1;  

24.         Object e = queue[parent];  

25.         if (comparator.compare(x, (E) e) >= 0)  

26.             break;  

27.         queue[k] = e;  

28.         k = parent;  

29.     }  

30.     queue[k] = x;  

31. }  
```



#### 5. 主要方法源码

##### 5.1 add

```
34. public boolean add(E e) {  

35.     return offer(e);  

36. }  


37. public boolean offer(E e) {  

38.     if (e == null)  

39.         throw new NullPointerException();  

40.     modCount++;  

41.     int i = size;  

42.     if (i >= queue.length)  

43.         grow(i + 1);  

44.     size = i + 1;  

45.     if (i == 0)  

46.         queue[0] = e;  

47.     else  

48.         siftUp(i, e);  

49.     return true;  

50. }  
```



增加过程免不了要扩容，PriortyQueue的扩容很简单，**如果当前容量比较小（小于64）的话进行双倍扩容，否则扩容50%**，源码如下：

```
1.  // 扩容函数  

2.  private void grow(int minCapacity) {  

3.      int oldCapacity = queue.length;  

4.      // 如果当前容量比较小（小于64）的话进行双倍扩容，否则扩容50%  

5.      int newCapacity = oldCapacity + ((oldCapacity < 64) ?  (oldCapacity + 2) : (oldCapacity >> 1));  

8.      // 如果发现扩容后溢出了，则进行调整  

9.      if (newCapacity - MAX_ARRAY_SIZE > 0)  

10.         newCapacity = hugeCapacity(minCapacity);  

11.     queue = Arrays.copyOf(queue, newCapacity);  

12. }  


13. // priorityQueue的最大容量  

14. private static int hugeCapacity(int minCapacity) {  

15.     if (minCapacity < 0) // overflow  

16.         throw new OutOfMemoryError();  

17.     return (minCapacity > MAX_ARRAY_SIZE) ?  

18.         Integer.MAX_VALUE :  

19.         MAX_ARRAY_SIZE;  

20. }  
```



因为priorityQueue是一个单向Queue，**先进先出**的原则，所以添加也只能在**尾部添加**，在尾部添加完之后需要调整，下面就是添加一个元素15的示意图：

![img](http://pcc.huitogo.club/c2f4c23dd0b4f835ff0d00cb3a7cbea4)

![img](http://pcc.huitogo.club/d09fbcbc143c8324f6cc9155356a24f1)

![img](http://pcc.huitogo.club/02f101f5c6ea3c477d0c6039731b7134)



##### 5.2 remove

删除元素分两种，一种是从顶部弹出元素，另一种是从数组中间删除元素

第一种源码如下：

```
1.  public E poll() {  

2.      if (size == 0)  

3.          return null;  

4.      int s = --size;  

5.      modCount++;  

6.      E result = (E) queue[0];  

7.      E x = (E) queue[s];  

8.      queue[s] = null;  

9.      if (s != 0)  

10.         siftDown(0, x);  

11.     return result;  

12. }  
```



从顶部弹出元素的示例图如下：

![img](http://pcc.huitogo.club/caa297f18a23f2991e9bb5c19e6acec4)

![img](http://pcc.huitogo.club/67b8b99a73610ec53aa49c6f65499cd6)

![img](http://pcc.huitogo.club/a10b28f65f7d45462d7ec9bd629e2d1c)

![img](http://pcc.huitogo.club/572d8a31d8beabde5c37083aec4ef303)

原理就是**用最后的元素当替补，然后再从上往下进行调整**



第二种源码如下：

```
1.  // 这里不是移除堆顶元素，而是移除指定元素  

2.  public boolean remove(Object o) {  

3.      // 先找到该元素的位置  

4.      int i = indexOf(o);  

5.      if (i == -1)  

6.          return false;  

7.      else {  

8.          removeAt(i);  

9.          return true;  

10.     }  

11. }  


12. // 移除指定序号的元素  

13. private E removeAt(int i) {  

14.     // assert i >= 0 && i < size;  

15.     modCount++;  

16.     // s为最后一个元素的序号  

17.     int s = --size;  

18.     if (s == i)   

19.         queue[i] = null;  

20.     else {  

21.         // moved记录最后一个元素的值  

22.         E moved = (E) queue[s];  

23.         queue[s] = null;  

24.         // 用最后一个元素代替要移除的元素，并向下进行调整  

25.         siftDown(i, moved);  

26.         // 如果向下调整后发现moved还在该位置，则再向上进行调整  

27.         if (queue[i] == moved) {  

28.             siftUp(i, moved);  

29.             if (queue[i] != moved)  

30.                 return moved;  

31.         }  

32.     }  

33.     return null;  

34. }  
```

原理和第一个差不多，也是**用最后一个元素做替代，然后先从上往下调整，如果没有改动就再从下往上调整**。



##### 5.3 indexof

按元素查找下标：

```
1.  private int indexOf(Object o) {  

2.      if (o != null) {  

3.          // 查找时需要进行全局遍历，比搜索二叉树的查找效率要低  

4.          for (int i = 0; i < size; i++)  

5.              if (o.equals(queue[i]))  

6.                  return i;  

7.      }  

8.      return -1;  

9.  }  
```



查找元素是否存在：

```
1.  public boolean contains(Object o) {  

2.      return indexOf(o) != -1;  

3.  }  
```



#### 6. PriorityQueue的应用场景

由于内部是用数组实现的小顶堆，所以堆适用的场景它都适用，比如典型的从n个元素中取出最小（最大）的前k个，这样的场景适用PriorityQueue就能以比较小的空间代价和还算ok的时间代价进行实现。

另一种就是需要动态插入元素，并且元素有优先级，需要根据一定的规则进行优先级排序。比如任务队列，队列动态插入，后面的任务优先级高的需要被先执行，那么使用优先级队列就可以比较好的实现这样的需求。