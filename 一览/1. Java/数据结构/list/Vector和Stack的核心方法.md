#### 1. Vector概述

![img](http://pcc.huitogo.club/3e809b5c8dd4930d0705e12ebf1a81e0)



通过API中可以知道：

1. Vector是一个可变化长度的数组
2. Vector增加长度通过的是capacity和capacityIncrement这两个变量
3. Vector也可以获得iterator和listIterator这两个迭代器，并且他们发生的是fail-fast，而不是fail-safe
4. Vector是一个线程安全的类，如果使用需要线程安全就使用Vector，如果不需要，就使用arrayList
5. Vector和ArrayList很类似，就少许的不一样，从它继承的类和实现的接口来看，跟arrayList一模一样。

**Verctor继承关系和ArrayList一模一样**



#### 2. Vector构造方法

##### 2.1 Vector() 

空构造



##### 2.2 Vector(int) 

带初始大小的构造

```
4.  public Vector(int initialCapacity) {  

5.      this(initialCapacity, 0);  

6.  }  
```



##### 2.3 Vector(int，int) 

带初始大小和增长大小的构造

```
8.  public Vector(int initialCapacity, int capacityIncrement) {  

9.      super();  

10.     if (initialCapacity < 0) //小于0，会报非法参数异常  

11.         throw new IllegalArgumentException("Illegal Capacity: "+   initialCapacity);  

13.     this.elementData = new Object[initialCapacity]; //初始化elementData数组  

14.     this.capacityIncrement = capacityIncrement; // 初始化elementData的增长大小，为0时增长一倍  

15. }  
```



##### 2.4 Vector(Collection<? extends E> c) 

带初始元素的构造

```
17. public Vector(Collection<? extends E> c) {  

18.     elementData = c.toArray();  

19.     elementCount = elementData.length;  

20.     // c.toArray might (incorrectly) not return Object[] (see 6260652)  

21.     if (elementData.getClass() != Object[].class)  

22.         elementData = Arrays.copyOf(elementData, elementCount, Object[].class);  

23. }  
```



#### 3. Vector核心方法

##### 3.1 add方法

```
26. //在vector中的末尾追加元素，利用synchronized实现线程安全。  

27. public synchronized boolean add(E e) {  

28.     modCount++;  

29.     //在增加元素前，检查容量是否够用  

30.     ensureCapacityHelper(elementCount + 1);  

31.     elementData[elementCount++] = e;  

32.     return true;  

33. }  
```



ensureCapacityHelper(elementCount + 1)代码

```
1.  //注意，这个方法是异步的  

2.  private void ensureCapacityHelper(int minCapacity) {  

3.      // overflow-conscious code  

4.      if (minCapacity - elementData.length > 0)  

5.          // 扩增，核心方法  

6.          grow(minCapacity);  

7.  } 
```



grow(minCapacity)代码

```
1.  // 这个方法跟arrayList一样，唯一的不同就是扩增大小  

2.  // 在capacityIncrement 为0时，在原有基础上扩展一倍  

3.  // 在capacityIncrement 不为0时，在原有基础上增长capacityIncrement  

4.  private void grow(int minCapacity) {  

5.      // overflow-conscious code  

6.      int oldCapacity = elementData.length;  

7.      int newCapacity = oldCapacity + ((capacityIncrement > 0) ?    capacityIncrement : oldCapacity);  

9.      if (newCapacity - minCapacity < 0)  

10.         newCapacity = minCapacity;  

11.     if (newCapacity - MAX_ARRAY_SIZE > 0)  

12.         newCapacity = hugeCapacity(minCapacity);  

13.     elementData = Arrays.copyOf(elementData, newCapacity);  

14. }  
```



其他的方法跟ArrayList类似，可以参考ArrayList的源码分析，只是为了实现线程安全，在一些操作elementData数组的方法上加了synchronized关键字实现同步，核心就是对elementData进行同步操作，如下：

```
1.  public synchronized E get(int index) {  

2.      if (index >= elementCount)  

3.          throw new ArrayIndexOutOfBoundsException(index);  

4.      return elementData(index);  

5.  } 
```



**Q1：vector是线程安全的，为什么在遍历中还可以进行增删操作？**

**因为在迭代器跌代过程中没有使用增删改查的那些加锁的方法，自然可以在迭代的时候修改**



**Q2：为什么不提倡使用Vector?**

**Vector在你不需要进行线程安全的时候，也会给你加锁，也就导致了额外开销**，所以在jdk1.5之后就被弃用了，现在如果要用到线程安全的集合，都是从java.util.concurrent包下去拿相应的类。



#### 4. Stack概述

![img](http://pcc.huitogo.club/291b7b313658119043ff86a4f18fc20a)

可以看出Stack是Vector的子类，相当于在Vector的基础上加了特有的方法，入栈、出栈等。

**底层维护的也是一个数组，父类中的构造，跟父类一样的扩增方式，并且它的方法也是同步的，所以也是线程安全。**



#### 5. 总结Vector和Stack

##### 5.1 Vector总结

1. Vector线程安全是因为它的方法都加了synchronized关键字
2. Vector的本质是一个数组，特点能是能够自动扩增，扩增的方式跟capacityIncrement的值有关
3. 它也会fail-fast



##### 5.2 Stack的总结

1. 对栈的一些操作，先进后出
2. 底层也是用数组实现的，因为继承了Vector
3. 也是线程安全的