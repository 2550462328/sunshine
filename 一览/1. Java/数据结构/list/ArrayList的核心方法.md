#### 1. ArrayList的概述

1. **ArrayList是可以动态增长和缩减的索引序列，它是基于数组实现的List类**。我们对ArrayList类的实例的所有的操作底层都是基于数组的。
2. 该类封装了一个动态再分配的Object[]数组，每一个类对象都有一个capacity属性，表示它们所封装的Object[]数组的长度，当向ArrayList中添加元素时，该属性值会自动增加，如果向ArrayList中**添加大量元素**，可使用ensureCapacity方法**一次性增加**capacity，可以减少增加重分配的次数**提高性能**。
3. ArrayList和Vector的区别是：ArrayList是线程不安全的，当多条线程访问同一个ArrayList集合时，程序需要手动保证该集合的同步性，而Vector则是线程安全的。

4. ArrayList和Collection的关系

![img](http://pcc.huitogo.club/1669175a6a1385a593b5f92d9765171c)



#### 2. ArrayList继承结构和层次关系

![img](http://pcc.huitogo.club/ae3d57544fa5aae08f8875d94ce23dc0)



**2.1 为什么先继承AbstractList，而让AbstractList先实现List<E>？而不是让ArrayList直接实现List<E>？**

**接口中全都是抽象的方法（jdk1.8后的default方法除外），而抽象类中可以有抽象方法，还可以有具体的实现方法**，正是利用了这一点，让AbstractList实现接口中一些通用的方法，而具体的类，如ArrayList就继承这个AbstractList类，拿到一些通用的方法，然后自己在实现一些自己特有的方法



**2.2 ArrayList实现了哪些接口？**

1） List<E>接口

ArrayList的父类AbstractList也实现了List<E>接口，那为什么子类ArrayList还是去实现一遍呢？

网址贴出来

[http://stackoverflow.com/questions/2165204/why-does-linkedhashsete-extend-hashsete-and-implement-sete]

开发这个collection 的作者Josh说。

这其实是一个**mistake，因为他写这代码的时候觉得这个会有用处，但是其实并没什么用**，但因为没什么影响，就一直留到了现在。



2）RandomAccess接口

这个是一个**标记性接口**，它的作用就是用来快速随机存取，有关效率的问题，在实现了该接口的话，那么使用普通的for循环来遍历，性能更高，例如arrayList。

而没有实现该接口的话，使用Iterator来迭代，这样性能更高，例如linkedList。所以这个标记性只是为了让我们知道我们用什么样的方式去获取数据性能更好。



3） Cloneable接口

实现了该接口，就可以使用Object.Clone()方法了。



4）Serializable接口

实现该序列化接口，表明该类可以被序列化



#### 3. 类属性

```
1.  // 版本号  

2.  private static final long serialVersionUID = 8683452581122892189L;  

3.  // 缺省容量  

4.  private static final int DEFAULT_CAPACITY = 10;  

5.  // 空对象数组  

6.  private static final Object[] EMPTY_ELEMENTDATA = {};  

7.  // 缺省空对象数组  

8.  private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};  

9.  // 元素数组  

10. transient Object[] elementData;  

11. // 实际元素大小，默认为0  

12. private int size;  

13. // 最大数组容量  

14. private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;  
```



在父类AbstractList中有个重要属性**modCount**，主要用于迭代的fast-fail。

在使用迭代器对list迭代的过程中，会进行判断。

```
1.  final void checkForComodification() {  

2.      if (modCount != expectedModCount)  

3.          throw new ConcurrentModificationException();  

4.  }  
```

这里的modCount来源于list，是数组修改的次数，发生add()、remove()的时候都会修改这个值，expectedModCount初始化赋值是modCount，但是list中没有方法会去更改这个expectedModCount，所以是一个固定值。所以当list发生修改操作的时候，modCount会改变，expectedModCount不会改变，所以上面方法检查后会报异常，所以**在list进行迭代过程中不可以对list进行add、remove操作**。



#### 4. 构造方法

##### 4.1 无参构造方法

```
1.  public ArrayList() {　　  

2.       super();     //调用父类中的无参构造方法，父类中的是个空的构造方法  

3.       //EMPTY_ELEMENTDATA：是个空的Object[]， 将elementData初始化，elementData也是个Object[]类型。空的Object[]会给默认大小10  

4.       this.elementData = EMPTY_ELEMENTDATA  

5.   }  
```



##### 4.2 有参构造函数一

```
1.  public ArrayList(int initialCapacity) {  

2.      super(); //父类中空的构造方法  

3.      if (initialCapacity < 0)  //判断如果自定义大小的容量小于0，则报下面这个非法数据异常  

4.          throw new IllegalArgumentException("Illegal Capacity: "+  initialCapacity);  

6.      this.elementData = new Object[initialCapacity]; //将自定义的容量大小当成初始化elementData的大小  

7.  }  
```



##### 4.3 有参构造函数二（不常用）

```
9.  public ArrayList(Collection<? extends E> c) {  

10.    elementData = c.toArray();    //转换为数组  

11.    size = elementData.length;   //数组中的数据个数  

12.    // c.toArray might (incorrectly) not return Object[] (see 6260652)  

13.    // 每个集合的toarray()的实现方法不一样，所以需要判断一下，如果不是Object[].class类型，那么就需要使用ArrayList中的方法去改造一下。  

14.    if (elementData.getClass() != Object[].class)   

15.        elementData = Arrays.copyOf(elementData, size, Object[].class);  

16. }　　
```



总结：**arrayList的构造方法就做一件事情，就是初始化一下储存数据的容器，其实本质上就是一个数组，在其中就叫elementData。**



#### 5. 核心方法

##### 5.1 add方法

![img](http://pcc.huitogo.club/3d6bea0cd89a5c64b7bb4b238e1c6b05)



**1）boolean add(E)：默认直接在末尾添加元素**

```
2.  public boolean add(E e) {      

3.  //确定内部容量是否够了，size是数组中数据的个数，因为要添加一个元素，所以size+1，先判断size+1的这个个数数组能否放得下，就在这个方法中去判断是否数组.length是否够用了。  

4.      ensureCapacityInternal(size + 1);  // Increments modCount!!  

5.   //在数据中正确的位置上放上元素e，并且size++  

6.      elementData[size++] = e;  

7.      return true;  

8.  } 
```



ensureCapacityInternal(size + 1)代码

```
1.  private void ensureCapacityInternal(int minCapacity) {  

2.      if (elementData == EMPTY_ELEMENTDATA) { // 判断初始化的elementData是不是空的数组，也就是没有长度  

3.      // 因为如果是空的话，minCapacity=size+1；其实就是等于1，空的数组没有长度就存放不了  

4.      // 所以就将minCapacity变成10，也就是默认大小  

5.          minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);  

6.      }  

7.      //确认实际的容量，上面只是将minCapacity=10，这个方法就是真正的判断elementData是否够用  

8.      ensureExplicitCapacity(minCapacity);  

9.  }  
```



ensureExplicitCapacity(minCapacity)代码

```
1.  private void ensureExplicitCapacity(int minCapacity) {  

2.      modCount++;  

3.      // minCapacity如果大于了实际elementData的长度，就要增加elementData的length      

4.      if (minCapacity - elementData.length > 0)  

5.          //arrayList能自动扩展大小的关键方法就在这里了  

6.          grow(minCapacity);  

7.      }  

8.  }  
```



**Q1：minCapacity到底是什么？**

**就是增加elementData容量所需要到达的最小值，minCapacity = size + 1**

第一种情况：由于elementData初始化时是空的数组，那么第一次add的时候，minCapacity=size+1；也就minCapacity=1，在上一个方法（确定内部容量ensureCapacityInternal）就会判断出是空的数组，就会给将minCapacity=10，到这一步为止，还没有改变elementData的大小。

第二种情况：elementData不是空的数组了，那么在add的时候，minCapacity=size+1；也就是minCapacity代表着elementData中增加之后的实际数据个数，拿着它判断elementData的length是否够用，如果length 不够用，那么肯定要扩大容量，不然增加的这个元素就会溢出。



grow(minCapacity)代码

```
1.  private void grow(int minCapacity) {  

2.       int oldCapacity = elementData.length;  //将扩充前的elementData大小给oldCapacity  

3.       int newCapacity = oldCapacity + (oldCapacity >> 1);//newCapacity就是1.5倍的oldCapacity  

4.       if (newCapacity - minCapacity < 0)//这句话就是适用于elementData就空数组的时候，length=0，那么oldCapacity=0，newCapacity=0，所以这个判断成立，在这里就是真正的初始化elementData的大小了，就是为10  

5.           newCapacity = minCapacity;  

6.       if (newCapacity - MAX_ARRAY_SIZE > 0)//如果newCapacity超过了最大的容量限制，就调用hugeCapacity，也就是将能给的最大值给newCapacity  

7.           newCapacity = hugeCapacity(minCapacity);  

8.       //新的容量大小已经确定好了，就copy数组，改变容量大小。  

9.       elementData = Arrays.copyOf(elementData, newCapacity);  

10.  }  
```



hugeCapacity(minCapacity)代码

```
1.  private static int hugeCapacity(int minCapacity) {  

2.      if (minCapacity < 0) // overflow  

3.          throw new OutOfMemoryError();  

4.      return (minCapacity > MAX_ARRAY_SIZE) ?   Integer.MAX_VALUE :  MAX_ARRAY_SIZE;  

7.  }  
```



**2）void add(int，E)：在特定位置添加元素，也就是插入元素**

```
9.  public void add(int index, E element) {  

10.     rangeCheckForAdd(index);//检查index也就是插入的位置是否合理。  

11.     ensureCapacityInternal(size + 1);  // Increments modCount!!  

12.     //这个方法就是用来在插入元素之后，要将index之后的元素都往后移一位，  

13.     System.arraycopy(elementData, index, elementData, index + 1,  size - index);  

15.     //在目标位置上存放元素  

16.     elementData[index] = element;  

17.     size++;//size增加1  

18. }   
```

System.arraycopy(...)：将elementData中从index开始size-index长度的元素复制到elementData从index+1的元素中。



总结add方法：

正常情况下会扩容1.5倍，特殊情况下（新扩展数组大小已经达到了最大值）则只取最大值。



举例说明一：

```
1.  List<Integer> lists = new ArrayList<Integer>();  

2.  lists.add(8); 
```



下图给出了未指定初始长度时最初与最后的elementData的大小。

![img](http://pcc.huitogo.club/26cc3666c10ba900c9e25d224ffa4f3b)

在add方法之前开始elementData ={}；调用add方法时会继续调用，直至grow，最后elementData的大小变为10，之后再返回到add函数，把8放在elementData[0]中



举例说明二：

```
1.  List<Integer> lists = new ArrayList<Integer>(6);  

2.  lists.add(8);  
```



下图是给定初始长度时最初和最后的elementData的大小

![img](http://pcc.huitogo.club/794e478e0644ce6efee91037a3bb1da2)

在调用add方法之前，elementData的大小已经为6，之后再进行传递，不会进行扩容处理



##### 5.2 remove方法

![img](http://pcc.huitogo.club/46fc3ec8dc06b375af946c78d96839c0)



**1）remove(int)：通过删除指定位置上的元素**

```
2.  public E remove(int index) {  

3.      rangeCheck(index);//检查index的合理性  

4.      modCount++;//这个作用很多，比如用来检测快速失败的一种标志。  

5.      E oldValue = elementData(index);//通过索引直接找到该元素  

6.      int numMoved = size - index - 1;//计算要移动的位数，如果在末尾remove就不需要移动  

7.      if (numMoved > 0)  

8.          System.arraycopy(elementData, index+1, elementData, index,   numMoved);  

10.     //将--size上的位置赋值为null，让gc(垃圾回收机制)更快的回收它。  

11.     elementData[--size] = null;   

12.     //返回删除的元素。  

13.     return oldValue;  

14. }  
```



**2）remove(Object)：这个方法可以看出来，arrayList是可以存放null值的**

```
16.  //fastRemove(index)方法的内部跟remove(index)的实现几乎一样，这里最主要是知道arrayList可以存储null值  

17.  public boolean remove(Object o) {  

18.     if (o == null) {  

19.         for (int index = 0; index < size; index++)  

20.             if (elementData[index] == null) {  

21.                 fastRemove(index);  

22.                 return true;  

23.             }  

24.     } else {  

25.         for (int index = 0; index < size; index++)  

26.             if (o.equals(elementData[index])) {  

27.                 fastRemove(index);  

28.                 return true;  

29.             }  

30.     }  

31.     return false;  

32. }  
```



**3）clear()：将elementData中每个元素都赋值为null，等待垃圾回收将这个给回收掉，所以叫clear**

```
34. public void clear() {  

35.      modCount++;  

36.      // clear to let GC do its work  

37.      for (int i = 0; i < size; i++)  

38.          elementData[i] = null;  

39.      size = 0;  

40.  }  
```



**4）removeAll(collection c)**

```
42. public boolean removeAll(Collection<?> c) {  

43.     return batchRemove(c, false);//批量删除  

44. }  
```



batchRemove(Collection, Boolean)：用于两个方法，一个removeAll()：它只清楚指定集合中的元素，这时传入的Boolean值为false；retainAll()用来测试两个集合是否有交集，这时传入的Boolean值为true。

```
1.  private boolean batchRemove(Collection<?> c, boolean complement) {  

2.       final Object[] elementData = this.elementData; //记录原集合A  

3.       int r = 0, w = 0;   

4.       boolean modified = false;  // 是否移除了数据，也就是batchRemove的返回值  

5.       try {  

6.           for (; r < size; r++)  

7.               // 分两种情形  

8.               // complement = true时，也就是retainAll方法，这时在集合A中记录交集，w为交集个数  

9.               // complement = false时，也就是removeAll方法，这时在集合A中记录集合c中没有的值，w为剩余元素个数  

10.              if (c.contains(elementData[r]) == complement)  

11.                  elementData[w++] = elementData[r];  

12.      } finally {  

13.          // Preserve behavioral compatibility with AbstractCollection,  

14.          // even if c.contains() throws.  

15.          //理论上循环结束时r=size，这里是处理在循环过程中出现异常的情况  

16.          if (r != size) {  

17.              //将剩下的元素都赋值给集合A，  

18.              System.arraycopy(elementData, r,  elementData, w,   size - r);  

21.              w += size - r;  

22.          }  

23.          // 这里是处理记录完交集或者不相关的值后，对于w和size之间这段元素的回收  

24.          if (w != size) {  

25.              // clear to let GC do its work  

26.              for (int i = w; i < size; i++)  

27.                  elementData[i] = null;  

28.              modCount += size - w;  

29.              size = w;  

30.              // 当w != size时，就以为着有交集或者有元素相等  

31.              modified = true;  

32.          }  

33.      }  

34.      return modified;  

35.  }  
```



总结：用户使用remove方法移除指定下标的元素，此时会把指定下标到数组末尾的元素向前移动一个单位，并且会把数组最后一个元素设置为null，这样是为了方便之后将整个数组不被使用时，会被GC，可以作为小的技巧使用。



##### 5.3 set方法

```
2.  public E set(int index, E element) {  

3.      // 检验索引是否合法  

4.      rangeCheck(index);  

5.      // 旧值  

6.      E oldValue = elementData(index);  

7.      // 赋新值  

8.      elementData[index] = element;  

9.      // 返回旧值  

10.     return oldValue;  

11. } 
```



##### 5.4 查找方法

**1）indexOf()方法**

```
14. // 从首开始查找数组里面是否存在指定元素  

15. public int indexOf(Object o) {  

16.     if (o == null) { // 查找的元素为空  

17.         for (int i = 0; i < size; i++) // 遍历数组，找到第一个为空的元素，返回下标  

18.             if (elementData[i]==null)  

19.                 return i;  

20.     } else { // 查找的元素不为空  

21.         for (int i = 0; i < size; i++) // 遍历数组，找到第一个和指定元素相等的元素，返回下标  

22.             if (o.equals(elementData[i]))  

23.                 return i;  

24.     }   

25.     // 没有找到，返回空  

26.     return -1;  

27. }  
```

说明：从头开始查找与指定元素相等的元素，注意是可以查找null元素的，意味着ArrayList中可以存放null元素的。与此函数对应的lastIndexOf，表示从尾部开始查找。



**2）get()方法**

```
2.  public E get(int index) {  

3.      // 检验索引是否合法  

4.      rangeCheck(index);  

5.      return elementData(index);  

6.  }  
```

说明：这里来使用的是**elementData()方法**返回元素值，不是直接从elementData数组中获取；**返回的值都经过了向下转型（Object -> E）**，这些是对我们应用程序屏蔽的小细节。



#### 6. 总结

1. arrayList可以存放null。
2. arrayList本质上就是一个elementData数组。
3. arrayList区别于数组的地方在于能够自动扩展大小，其中关键的方法就是grow()方法。
4. arrayList中removeAll(collection c)和clear()的区别就是removeAll可以删除批量指定的元素，而clear是全是删除集合中的元素。
5. arrayList由于本质是数组，所以它在数据的查询方面会很快，而在插入删除这些方面，性能下降很多，有移动很多数据才能达到应有的效果。
6. arrayList实现了RandomAccess，所以在遍历它的时候推荐使用for循环。