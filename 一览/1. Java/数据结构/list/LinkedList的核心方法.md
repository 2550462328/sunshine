### 1. LinkedList概述

LinkedList的继承关系如图所示

![img](http://pcc.huitogo.club/f2faa5d61f235ae45e9421e69d7e7ca9)



LinkedList是一种**可以在任何位置进行高效地插入和移除操作的有序序列**，它是**基于双向链表**实现的。

LinkedList 是一个继承于AbstractSequentialList的双向链表。它也可以被当作堆栈、队列或双端队列进行操作。

LinkedList 实现 List 接口，能对它进行队列操作。

LinkedList 实现 Deque 接口，即能将LinkedList当作**双端队列使用**。

LinkedList 实现了Cloneable接口，即覆盖了函数clone()，能克隆。

LinkedList 实现java.io.Serializable接口，这意味着LinkedList支持序列化，能通过序列化去传输。

LinkedList 是非同步的。



注意：LinkedList没有实现RandomAccess接口，所以不是随机存取，意味着**在遍历LinkedList的时候，使用Iterator遍历效率更高，\**也可以使用\**foreach增强循环，原理也是使用Iterator**！



**Q1：为什么LinkedList和AbstractList之间多了一个AbstractSequentialList抽象类？**

减少实现顺序存取（例如LinkedList）这种类的工作，通俗的讲就是方便，抽象出类似LinkedList这种类的一些共同的方法

以后如果自己想实现**顺序存取**这种特性的类(就是链表形式)，那么就继承这个**AbstractSequentialList抽象类**，如果想像数组那样的**随机存取**的类，那么就去实现**AbstracList抽象类**



### 2. LinkedList的数据结构

#### 2.1 单向链表

element：用来存放元素

next：用来指向下一个节点元素

**通过每个结点的指针指向下一个结点从而链接起来的结构，最后一个节点的next指向null。**

![img](http://pcc.huitogo.club/300caf64c8cf7ea2175a68a3da31404f)



特殊：单向循环链表

在单向链表的最后一个节点的next会指向头节点，而不是指向null，这样存成一个环

![img](http://pcc.huitogo.club/7519e770e0e081c81a75396139f98ef9)



#### 2.2 双向链表

element：存放元素

pre：用来指向前一个元素

next：指向后一个元素

双向链表是包含**两个指针的，pre指向前一个节点，next指向后一个节点，但是第一个节点head的pre指向null，最后一个节点的tail指向null**

![img](http://pcc.huitogo.club/6fa395efd43f363e4145cf69c6a0307a)



特殊：双向循环链表

**第一个节点的pre指向最后一个节点，最后一个节点的next指向第一个节点，也形成一个“环”**

![img](http://pcc.huitogo.club/b58dc53d0147e96fea8c6cea16466ba5)



#### 2.3  LinkedList

![img](http://pcc.huitogo.club/af23638c59baac6019cb49b98eb0e2dc)

LinkedList底层使用的**双向链表结构**，有一个头结点和一个尾结点，双向链表意味着我们可以从头开始正向遍历，或者是从尾开始逆向遍历，并且可以针对头部和尾部进行相应的操作。



### 3. 类属性

```
2.  // 实际元素个数  

3.  transient int size = 0;  

4.  // 头结点  

5.  transient Node<E> first;  

6.  // 尾结点  

7.  transient Node<E> last; 
```

头结点、尾结点都有transient关键字修饰，这也意味着在序列化时该域是不会序列化的



### 4. 构造方法

#### 4.1 空参构造函数

```
3.  /** 

4.   * Constructs an empty list. 

5.   */  

6.  public LinkedList() {}  
```



#### 4.2 有参构造函数

```
8.  //将集合c中的各个元素构建成LinkedList链表。  

9.   public LinkedList(Collection<? extends E> c) {  

10.   // 调用无参构造函数  

11.      this();  

12.      // 添加集合中所有的元素  

13.      addAll(c);  

14.  }  
```



链表的核心是Node节点，用于存放实际元素的地方

```
1.  private static class Node<E> {  

2.      E item; // 数据域（当前节点的值）  

3.      Node<E> next; // 后继（指向当前一个节点的后一个节点）  

4.      Node<E> prev; // 前驱（指向当前节点的前一个节点）  


5.      // 构造函数，赋值前驱后继  

6.      Node(Node<E> prev, E element, Node<E> next) {  

7.          this.item = element;  

8.          this.next = next;  

9.          this.prev = prev;  

10.     }  

11. }  
```



### 5. LinkedList的核心方法

方法实现的核心理念就是**操作节点的pre、next，打乱链表的顺序重新串起来。**

#### 5.1 add方法

![img](http://pcc.huitogo.club/0ba7b86441bd5f874e69f18f769a17e4)



**1） add(E)**

```
2.  public boolean add(E e) {  

3.     　　 // 添加到末尾  

4.      　　linkLast(e);  

5.      　　return true;  

6.   }
```



linkLast(e)代码

```
1.  // 向尾部添加节点  

2.  void linkLast(E e) {  

3.      final Node<E> l = last;     

4.      final Node<E> newNode = new Node<>(l, e, null); //将e封装为节点，并且e.prev指向了最后一个节点  

5.      last = newNode; //从尾部添加，所以newNode是最后一个节点   

6.      if (l == null)    //当前链表为控制，newNode既是最后一个节点也是第一个节点  

7.          first = newNode;  

8.      else      

9.          l.next = newNode; // 添加到末尾  

10.     size++; //添加一个节点，size自增  

11.     modCount++;  

12. }  
```



下面是空LinkedList添加元素的结构图：

![img](http://pcc.huitogo.club/6cd227cbbf0bccaa24d0f813ab40951c)



**2） addAll(c)**

```
2.  public boolean addAll(Collection\<? extends E\> c) {  

3.      return addAll(size, c);  

4.  }  
```



addAll(size, c)，核心代码

```
1.  public boolean addAll(int index, Collection<? extends E> c) {  

2.      // 检查index  

3.      checkPositionIndex(index);  

4.      //转换为Object数组  

5.      Object[] a = c.toArray();  

6.      int numNew = a.length;  

7.      if (numNew == 0)  

8.          return false;  

9.      // pred是添加每一个元素时的前一个节点，succ是后一个节点  

10.     Node<E> pred, succ;  

11.     // 当前链表为空或者在末尾添加元素  

12.     if (index == size) {  

13.         // 在这种情况下待添加元素的后一个节点为空，前一个节点等于最后一个节点  

14.         succ = null;  

15.         pred = last;  

16.     } else { 在链表中间插入元素  

17.         // 这种情况，待添加元素的后一个节点就是插入前该位置的节点，也就是node(index)  

18.         // 前一个节点自然就是之前位置节点的前一个节点  

19.         succ = node(index);  

20.         pred = succ.prev;  

21.     }  

22.     //遍历数组a中的元素，封装为一个个节点并插入链表中  

23.     for (Object o : a) {  

24.         @SuppressWarnings("unchecked") E e = (E) o;  

25.         // 封装节点  

26.         Node<E> newNode = new Node<>(pred, e, null);  

27.         if (pred == null) // 插入前链表为空，那么第一个节点就是newNode  

28.             first = newNode;  

29.         else  

30.             pred.next = newNode;  

31.         //将newNode变成pred，连接下一个节点  

32.         pred = newNode;  

33.     }  

34.     // 遍历完待插入的所有节点后，怎么定义最后一个节点  

35.     if (succ == null) { // 插入前链表为空或者在尾部添加节点  

36.         last = pred; // 上面遍历时将最后一个newNode变成了pred，所以这种情况最后一个newNode就是last  

37.     } else { //中间插入元素，succ是有值的，直接关联最后一个newNode即可  

38.         pred.next = succ;  

39.         succ.prev = pred;  

40.     }  

41.     //增加了几个元素，就把 size = size +numNew  

42.     size += numNew;  

43.     modCount++;  

44.     return true;  

45. } 
```



node函数是根据索引下标找到该结点并返回，参数中的index表示在索引下标为index的结点（实际上是第index + 1个结点）的前面插入

```
1.  Node<E> node(int index) {  

2.      // 判断插入的位置在链表前半段或者是后半段  

3.      if (index < (size >> 1)) { // 插入位置在前半段  

4.          Node<E> x = first;   

5.          for (int i = 0; i < index; i++) // 从头结点开始正向遍历  

6.              x = x.next;  

7.          return x; // 返回该结点  

8.      } else { // 插入位置在后半段  

9.          Node<E> x = last;   

10.         for (int i = size - 1; i > index; i--) // 从尾结点开始反向遍历  

11.             x = x.prev;  

12.         return x; // 返回该结点  

13.     }  

14. } 
```

这里有个小优化，节点在前半段则从头开始遍历，在后半段则从尾开始遍历，这样就保证了只需要遍历最多一半结点就可以找到指定索引的结点。



**Q1：在addAll(size,c)中遍历前为什么要将集合c转换成数组进行遍历？**

直接遍历 Collection，还要先生成它的 Iterator<E>，再遍历。倒不如，一次性转变成数组来遍历，更便捷



#### 5.2 remove方法

```
2.  public boolean remove(Object o) {  

3.  //这里可以看到，linkedList也能存储null  

4.      if (o == null) {  

5.          //循环遍历链表，直到找到null值，然后使用unlink移除该值。下面的这个else中也一样  

6.          for (Node<E> x = first; x != null; x = x.next) {  

7.              if (x.item == null) {  

8.                  unlink(x);  

9.                  return true;  

10.             }  

11.         }  

12.     } else {  

13.         for (Node<E> x = first; x != null; x = x.next) {  

14.             if (o.equals(x.item)) {  

15.                 unlink(x);  

16.                 return true;  

17.             }  

18.         }  

19.     }  

20.     return false;  

21. }  
```



unlink(x) 代码

```
1.  E unlink(Node<E> x) {  

2.      // assert x != null;  

3.      final E element = x.item;  

4.      final Node<E> next = x.next;  

5.      final Node<E> prev = x.prev;  

6.      if (prev == null) { //说明移除的节点是头节点  

7.          //first头节点应该指向下一个节点  

8.          first = next;  

9.      } else {  

10.         //不是头节点  

11.         prev.next = next;  

12.         //解除x节点的前指向，解除关联让它gc  

13.         x.prev = null;  

14.     }  

15.     if (next == null) { //说明移除的节点是尾节点  

16.         last = prev;  

17.     } else {  

18.         //不是尾节点，prev和next进行关联  

19.         next.prev = prev;  

20.         // 解除x节点的前指向，解除关联让它gc  

21.         x.next = null;  

22.     }  

23.     //x的前后指向都为null了，也把item为null，让gc回收它  

24.     x.item = null;  

25.     size--;    //移除一个节点，size自减  

26.     modCount++;  

27.     return element;    //由于一开始已经保存了x的值到element，所以返回。  

28. }  
```



#### 5.3 get方法

**1） get(index) 查询元素的方法**

```
31. public E get(int index) {  

32.     checkElementIndex(index);  

33.     return node(index).item;  

34. }  
```

调用的就是node(index)方法



**2）indexOf(Object o)**

```
2.  // 和remove代码类似，只是返回类型不一样  

3.  public int indexOf(Object o) {  

4.      int index = 0;  

5.      if (o == null) {  

6.          for (Node<E> x = first; x != null; x = x.next) {  

7.              if (x.item == null)  

8.                  return index;  

9.              index++;  

10.         }  

11.     } else {  

12.         for (Node<E> x = first; x != null; x = x.next) {  

13.             if (o.equals(x.item))  

14.                 return index;  

15.             index++;  

16.         }  

17.     }  

18.     return -1;  

19. }  
```



### 6. LinkedList的迭代器

在LinkedList中除了有一个Node的内部类外，应该还能看到另外两个内部类，那就是ListItr，还有一个是DescendingIterator。

#### 6.1 ListItr内部类

ListItr继承了ListIterator

![img](http://pcc.huitogo.club/6af1f29da0e808b3bfb7602f949e9c1c)

可以看出ListItr让linkedList**不光能像后迭代，也能向前迭代**而且在迭代的过程中，还能**移除、修改、添加值**的操作。



#### 6.2 DescendingIterator内部类

```
2.  private class DescendingIterator implements Iterator<E> {  

3.      private final ListItr itr = new ListItr(size());  

4.      public boolean hasNext() {  

5.          return itr.hasPrevious();  

6.      }  

7.      public E next() {  

8.          return itr.previous();  

9.      }  

10.     public void remove() {  

11.         itr.remove();  

12.     }  

13. }  
```

这个类**还是调用的ListItr**，作用是**封装一下Itr中几个方法**，**让使用者以正常的思维去写代码**，例如，在从后往前遍历的时候，也是跟从前往后遍历一样，使用next等操作，而不用使用特殊的previous。



### 7. 总结

1. linkedList本质上是一个双向链表，通过一个Node内部类实现的这种链表结构。
2. 能存储null值
3. 跟arrayList相比较，就真正的知道了，LinkedList在删除和增加等操作上性能好，而ArrayList在查询的性能上好
4. 从源码中看，它不存在容量不足的情况
5. linkedList不光能够向前迭代，还能像后迭代，并且在迭代的过程中，可以修改值、添加值、还能移除值。
6. linkedList不光能当链表，还能当队列使用，这个就是因为实现了Deque接口。