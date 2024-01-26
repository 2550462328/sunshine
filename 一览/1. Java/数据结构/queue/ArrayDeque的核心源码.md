#### 1. ArrayDeque介绍

1. 从 ArrayDeque 命名就能看出他的实现基于**可扩容动态数组**，每次队列满了就会进行扩容，除非扩容至 int 边界才会抛出异常
2. **ArrayDeque 不允许元素为 null**
3. ArrayDeque 的主要成员是一个 elements 数组和 int 的 head 与 tail 索引，head 是队列的头部元素索引，而 tail 是队列下一个要添加的元素的索引，**elements的默认容量是 16 且默认容量必须是 2 的幂次，不足 2 的幂次会自动向上调整为 2的幂次**
4. ArrayDeque可以高效的进行**元素查找和尾部插入取出**，是用作队列、双端队列、栈的绝佳选择，**性能比LinkedList还要好**



#### 2. ArrayDeque的结构

ArrayDeque的整体继承结构如下：

![img](http://pcc.huitogo.club/dd7a502ee462bba2b4baf16e16eb5157)



ArrayDeque的内部结构元素如下：

```
1.  //存储元素的数组  

2.  transient Object[] elements; // 非private访问限制，以便内部类访问  

3.   /** 

4.    * 头部节点序号 

5.    */  

6.   transient int head;  

7.   /** 

8.    * 尾部节点序号，（指向最后一点节点的后一个位置） 

9.    */  

10.  transient int tail;  

11.  /** 

12.   * 双端队列的最小容量，必须是2的幂 

13.   */  

14.  private static final int MIN_INITIAL_CAPACITY = 8;  
```



#### 3. ArrayDeque的常用方法

![img](http://pcc.huitogo.club/bac373f510c77fc094888a5f3a95e473)



1. ArrayDeque 获取队列头部元素的 element()、getFirst()、peek()、peekFirst()操作，其都是调用 getFirst() 实现的，**访问队列头部元素但不删除**
2. ArrayDeque 删除队列头部元素的 remove()、removeFirst()、poll()、pollFirst()操作，其都是调用 pollFirst() 实现的，**移除队列头部元素且返回被移除的元素**
3. ArrayDeque 添加元素到队列尾部的操作可以发现 add(E e)、offer(E e)、offerLast(E e)、addLast(E e) 操作都是调用 addLast(E e) 实现的



**Q1：为什么会有这么多方法？**

其实主要是为了模拟不同的数据结构，如栈操作：pop，push，peek，队列操作：add，offer，remove，poll，peek，element，双端队列操作：addFirst，addLast，getFirst，getLast，peekFirst，peekLast，removeFirst，removeLast，pollFirst，pollLast。



#### 4. 为什么说ArrayDeque是循环数组？

传统的非循环数组中，在尾部添加元素的时候如果添加元素下标等于数组长度那么就要进行扩容操作，但是，在ArrayDeque中是一个双端队列，我们操作ArrayDeque的物理逻辑如下：

![img](http://pcc.huitogo.club/0d60f4701417a6d8ce4360124545b5ce)



正如上图中最后的多次操作结果所示，如果此时我们再 add 操作一个元素到 tail索引处则 tail+1 会变成 8导致数组越界，理论上来说这时候应该进行扩容操作了，但是由于下标为 0、1、2、3 处没有存储元素，直接扩容有些浪费（**ArrayList为了避免浪费是通过拷贝将删除之后的元素整体前挪一位**）



现在看一下addLast的源码，是怎么处理这种情况的

```
1.  public void addLast(E e) {  

2.      if (e == null)  

3.          throw new NullPointerException();  

4.      elements[tail] = e;  

5.      if ( (tail = (tail + 1) & (elements.length - 1)) == head)  

6.          doubleCapacity();  

7.  } 
```

可以看到为了高效利用数组中现有的剩余空间就有了 addLast(E e) 中的代码 **(tail = (tail + 1) & (elements.length - 1))**;

如果让我们来写这个循环实现代码的话可能就是，判断一下有没有超过最大容量给tail赋值再判断一下head的位置，最后再扩容，这里用一个位运算解决了所有逻辑，可以说很精妙，在ArrayDeque的源码中有多处这样的边界判断，**这里也就是ArrayDeque循环数组的核心**



假设 elements 默认初始化长度是 8，则当前 tail +1（8=1000）按位与上数组长度减一（7=0111）的结果为十进制的 0，所以下一个被addLast(E e) 的元素实际会放在索引为 0 的位置，再下一个会放在索引为 1的位置，如下图：

![img](http://pcc.huitogo.club/a9774c4d3f5f81d21f0ef115195e7036)

**所以 ArrayDeque是一个双向循环队列，其基于数组实现了双向循环操作（依赖按位与操作循环），初始容量必须是2 的幂次，默认为 16，存储的元素不能为空，非线程并发安全。**



#### 5. ArrayDeque什么时候扩容，是怎么扩容的？

在上面addLast的源码中，以及ArrayDeque双向循环列队的原理，我们知道随着出队入队不断操作，如果 tail 移动到 length-1 之后数组的第一个位置 0 处没有元素则需要将 tail 指向 0，依次循环，**当 tail 如果等于 head 时说明数组要满了，接下来需要进行数组扩容**



扩容操作的源码如下：

```
1.  private void doubleCapacity() {  

2.      assert head == tail;  

3.      int p = head;  

4.      int n = elements.length;  

5.      int r = n - p;// number of elements to the right of p  

6.      // 新容量直接翻倍  

7.      int newCapacity = n << 1;  

8.      if (newCapacity < 0) throw new IllegalStateException("Sorry, deque too big");  

9.      Object[] a = new Object[newCapacity];  

10.     //扩容后分两次复制，head到当前数组末尾  

11.     System.arraycopy(elements, p, a, 0, r);  

12.     // 当前数组头到tail位置   

13.     System.arraycopy(elements, 0, a, r, p);  

14.     //相关索引置位  

15.     elements = a;  

16.     head = 0;  

17.     tail = n;  

18. }  
```



可以看到扩容也是按2的幂次方扩容的，先将head到数组末尾的数据复制到新数组的头部，将数组头到tail的部分复制到新数组的后面，上面的扩容操作流程演示如下：

![img](http://pcc.huitogo.club/4145ff77bfb468ecf199dc89ca910ed8)



#### 6. ArrayDeque的初始数组容量怎么定义的？扩容大小为什么必须是2的幂次方？

ArrayDeque的构造方法源码如下：

```
1.  public ArrayDeque(int numElements) {  

2.      allocateElements(numElements);  

3.  }  


4.  private void allocateElements(int numElements) {  

5.      elements = new Object[calculateSize(numElements)];  

6.  }  


7.  private static int calculateSize(int numElements) {  

8.      int initialCapacity = MIN_INITIAL_CAPACITY;  

9.      if (numElements >= initialCapacity) {  

10.         initialCapacity = numElements;  

11.         initialCapacity |= (initialCapacity >>>  1);  

12.         initialCapacity |= (initialCapacity >>>  2);  

13.         initialCapacity |= (initialCapacity >>>  4);  

14.         initialCapacity |= (initialCapacity >>>  8);  

15.         initialCapacity |= (initialCapacity >>> 16);  

16.         initialCapacity++;  

17.         if (initialCapacity < 0)   // Too many elements, must back off  

18.             initialCapacity >>>= 1;// Good luck allocating 2 ^ 30 elements  

19.     }  

20.     return initialCapacity;  

21. }  
```

这里重点需要关注的是calculateSize的源码，**将指定的容量计算成比它大的最小2的幂次方值**



先讨论一下这个计算的实现原理：

在Java中，int类型是占4字节，也就是32位。简单起见，这里以一个8位二进制数来演示前三次操作。假设这个二进制数对应的十进制为89，整个过程如下：

![img](http://pcc.huitogo.club/73a66363170cfdcf0bdf1aee6869d8f5)



可以看到最后，除了第一位，其他位全部变成了1，然后这个结果再加一，即得到大于89的最小的2次幂，也许你会想，为什么右移的数值要分别是1，2，4，8，16呢？

事实上，在这系列操作中，其他位只是配角，**我们只需要关注第一个不为0的位即可**，假设其为第n位，先右移一位然后进行或操作，得到的结果，第n位和第n-1位肯定为1，这样就有两个位为1了，然后进行右移两位，再进行或操作，那么第n位到第n-3位一定都为1，然后右移4位，依次类推。int为32位，因此，最后只需要移动16位即可。1+2+4+8+16 = 31，所以经过这一波操作，原数字对应的二进制，操作得到的结果将是从其第一个不为0的位开始，往后的位均为1。

在calculateSize中计算完比指定数大的最小2的幂次方数值后，还有一个需要考虑的就是如果扩容到了2^32 - 1，也就是int的最大值，这时候再加1，也就是超过int最大值怎么办？

```
1.  if (initialCapacity < 0)   

2.      initialCapacity >>>= 1;
```

第32位为1的情况下，第32位为符号位，1代表负数，这样的话就必须回退一步，将得到的数右移一位（即2 ^ 30）



知道calculateSize的计算原理后再看一下为什么是2的幂次方值？为什么扩容的时候也是 << 1？

**因为只有容量为 2 的幂次时 ((tail + 1) & (elements.length - 1)) 操作中的(elements.length - 1) 二进制最高位永远为 0，当 (tail + 1)与其按位与操作时才能保证循环归零置位。**



#### 7. ArrayDeque的增删改查源码？

增的话主要看addLast和addFirst就可以了，addLast源码上面有，这里看一下addFirst的源码：

```
1.  public void addFirst(E e) {  

2.       if (e == null)  

3.           throw new NullPointerException();  

4.       elements[head = (head - 1) & (elements.length - 1)] = e;  

5.       if (head == tail)  

6.           doubleCapacity();  

7.   }  
```

这里head = (head - 1) & (elements.length - 1)和addLast中的tail = (tail + 1) & (elements.length - 1)作用一样，都是实现循环数组的核心



查的话主要看一下getFirst和getLast，peekFirst和peekLast类似

```
1.  public E getFirst() {  

2.      @SuppressWarnings("unchecked")  

3.      E result = (E) elements[head];  

4.      if (result == null)  

5.          throw new NoSuchElementException();  

6.      return result;  

7.  }  


8.  public E getLast() {  

9.      @SuppressWarnings("unchecked")  

10.     E result = (E) elements[(tail - 1) & (elements.length - 1)];  

11.     if (result == null)  

12.         throw new NoSuchElementException();  

13.     return result;  

14. }      
```



删的话分两种

一种是头尾移除元素，主要看一下pollFirst和pollLast

```
1.  public E pollFirst() {  

2.      int h = head;  

3.      @SuppressWarnings("unchecked")  

4.      E result = (E) elements[h];  

5.      // 如果结果为null则返回null  

6.      if (result == null)  

7.          return null;  

8.      elements[h] = null;     // Must null out slot  

9.      head = (h + 1) & (elements.length - 1);  

10.     return result;  

11. }  


12. public E pollLast() {  

13.     int t = (tail - 1) & (elements.length - 1);  

14.     @SuppressWarnings("unchecked")  

15.     E result = (E) elements[t];  

16.     if (result == null)  

17.         return null;  

18.     elements[t] = null;  

19.     tail = t;  

20.     return result;  

21. }  
```



另一种就是在数组中间移除元素，这个就比较复杂了，ArrayDeque使用removeFirstOccurrence和removeLastOccurrence来实现从数组中移除第一个出现指定元素和最后一个出现指定元素的位置，源码如下：

```
1.  public boolean removeFirstOccurrence(Object o) {  

2.      if (o == null)  

3.          return false;  

4.      int mask = elements.length - 1;  

5.      int i = head;  

6.      Object x;  

7.      while ( (x = elements[i]) != null) {  

8.          if (o.equals(x)) {  

9.              delete(i);  

10.             return true;  

11.         }  

12.         i = (i + 1) & mask;  

13.     }  

14.     return false;  

15. }  


16. public boolean removeLastOccurrence(Object o) {  

17.     if (o == null)  

18.         return false;  

19.     int mask = elements.length - 1;  

20.     int i = (tail - 1) & mask;  

21.     Object x;  

22.     while ( (x = elements[i]) != null) {  

23.         if (o.equals(x)) {  

24.             delete(i);  

25.             return true;  

26.         }  

27.         i = (i - 1) & mask;  

28.     }  

29.     return false;  

30. }  
```

其实都是通过循环遍历的方式进行查找一个是从head开始往后查找，一个是从tail开始往前查找



主要使用到的是ArrayDeque的私有方法delete(int i)来删除中间元素，删除中间元素时，即使是循环数组，也需要批量移动数组元素了，所以删除中间元素实际上面临三个问题，一是需要在左侧或右侧删除，二是需要挪动头或尾，三是优化需要，尽量少得移动数组元素。 ArrayDeque实际上是先从第三个问题入手的，先判断中间元素里head近还是离tail近，然后移动较近的那一端。 不过，较近的一端只是逻辑上较近，物理数组上，可能被分成了两截，这就需要做**两次数组元素的批量移动**。

```
1.  private boolean delete(int i) {  

2.      // 先做不变性检测，判断是否当前结构满足删除需求  

3.      checkInvariants();  

4.      final Object[] elements = this.elements;  

5.      // mask即掩码  

6.      final int mask = elements.length - 1;  

7.      final int h = head;  

8.      final int t = tail;  

9.      // front代表i到头部的距离  

10.     final int front = (i - h) & mask;  

11.     // back代表i到尾部的距离  

12.     final int back  = (t - i) & mask;  

13.     // 再次校验，如果i到头部的距离大于等于尾部到头部的距离，表示当前队列已经被修改了，通过最开始检测后，i是不应该满足该条件的  

14.     if (front >= ((t - h) & mask))  

15.         throw new ConcurrentModificationException();  

16.     // 为移动尽量少的元素做优化，如果离头部比较近，则将该位置到头部的元素进行移动，如果离尾部比较近，则将该位置到尾部的元素进行移动  

17.     if (front < back) {  

18.         if (h <= i) {  

19.             System.arraycopy(elements, h, elements, h + 1, front);  

20.         } else { // Wrap around  

21.             System.arraycopy(elements, 0, elements, 1, i);  

22.             elements[0] = elements[mask];  

23.             System.arraycopy(elements, h, elements, h + 1, mask - h);  

24.         }  

25.         elements[h] = null;  

26.         head = (h + 1) & mask;  

27.         return false;  

28.     } else {  

29.         if (i < t) { // Copy the null tail as well  

30.             System.arraycopy(elements, i + 1, elements, i, back);  

31.             tail = t - 1;  

32.         } else { // Wrap around  

33.             System.arraycopy(elements, i + 1, elements, i, mask - i);  

34.             elements[mask] = elements[0];  

35.             System.arraycopy(elements, 1, elements, 0, t);  

36.             tail = (t - 1) & mask;  

37.         }  

38.         return true;  

39.     }  

40. }    
```

这个delete还是花了一点心思的，不仅做了两次校验，还对元素移动进行了优化



#### 8. ArrayDeque可以逆向遍历吗？

当然可以，ArrayDeque作为双向队列，有head和tail两个标识可以作为反向遍历的根本，源码如下：

```
1.  public Iterator<E> descendingIterator() {  

2.      return new DescendingIterator();  

3.  }  


4.  private class DescendingIterator implements Iterator<E> {  

5.      private int cursor = tail;  

6.      private int fence = head;  

7.      private int lastRet = -1;  

8.      public boolean hasNext() {  

9.          return cursor != fence;  

10.     }  

11.     public E next() {  

12.         if (cursor == fence)  

13.             throw new NoSuchElementException();  

14.         cursor = (cursor - 1) & (elements.length - 1);  

15.         @SuppressWarnings("unchecked")  

16.         E result = (E) elements[cursor];  

17.         // 如果移除了尾部元素，会导致 tail != fence  

18.         // 如果移除了头部元素，会导致 result == null  

19.         if (head != fence || result == null)   //这种检测比较弱，如果先移除一个尾部元素，然后再添加一个尾部元素，那么tail依旧和fence相等，这种情况就检测不出来了

20.             throw new ConcurrentModificationException();  

21.         lastRet = cursor;  

22.         return result;  

23.     }  


24.     public void remove() {  

25.         if (lastRet < 0)  

26.             throw new IllegalStateException();  

27.         if (!delete(lastRet)) {  

28.             cursor = (cursor + 1) & (elements.length - 1);  

29.             fence = head;  

30.         }  

31.         lastRet = -1;  

32.     }  

33. }  
```



#### 9. ArrayDeque是怎么计算队列长度的？

当tail > head的时候比较好计算，但是在tail < head的时候，也就是物理上元素不连续的时候怎么计算？

如果让我们自己写，可能也是分条件判断，分别计算两截的数据



但是ArrayDeque完美的运用位运算解决这个问题

```
1. return (tail - head) & (elements.length - 1); 
```

当物理上被分为两截时，tail-head会是负数，整个操作相当于取模运算，例如，当tail为3，head为14，物理数组长度16时，运算的就是11110101&00001111，值为00000101，也就是5。



#### 10. ArrayDeque怎么转换成数组的？

从基本操作上，如果物理上不连续，先复制右侧，再补上左侧。

toArray的问题在于新数组的长度是动态的，为了生成大小刚好的新数组，ArrayDeque使用了Arrays工具类来实现这个特定长度的数组，并同时实现对head一侧数据的复制：

toArray的源码如下：

```
1.  public Object[] toArray() {  

2.      return copyElements(new Object[size()]);  

3.  }  


4.  private <T> T[] copyElements(T[] a) {  

5.      if (head < tail) {  

6.          System.arraycopy(elements, head, a, 0, size());  

7.      } else if (head > tail) {  

8.          int headPortionLen = elements.length - head;  

9.          System.arraycopy(elements, head, a, 0, headPortionLen);  

10.         System.arraycopy(elements, 0, a, headPortionLen, tail);  

11.     }  

12.     return a;  

13. }  
```

这样的话，如果物理空间连续，就直接复制完成；

但如果物理空间不连续，第一次Arrays.copyOfRange需要保证一次性生成足够的物理空间，所以end的值不能是length（否则长度就只有length-head这么长），而应该是tail+length。



#### 11. 怎么看待ArrayDeque的存取和查询效率？

因为ArrayDeque底层维护的是一个数组，所以在查询方面效率高，在增删方面，在ArrayList中因为增删元素的话需要移动大量元素，所以效率比较低，但

是在ArrayDeque中如果在头尾增删的话不用移动，在中间增删的话，优化了移动的步骤，同时减少了System.arrayCopy方法的使用，所以ArrayDeque这个数据集合的性能非常优良。