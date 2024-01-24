简单的来说：在java.util下的集合都是发生fail-fast，而在java.util.concurrent下的发生的都是fail-safe。

**1. fail-fast**

例如在arrayList中使用迭代器**遍历时**，有另外的线程对arrayList的**存储数组进行了改变**，比如add、delete、等使之发生了结构上的改变，所以Iterator就会快速报一个java.util.ConcurrentModificationException异常（并发修改异常），这就是快速失败。



**2. fail-safe**

java.util.concurrent下的类，都是线程安全的类，他们在迭代的过程中，如果有线程进行结构的改变，不会报异常，而是正常遍历，这就是安全失败。



**Q1：为什么concurrent下的类在迭代中数据结构改变不会报异常？**

在concurrent下的集合类增加元素的时候使用Arrays.copyOf()来拷贝副本，**在副本上增加元素**，如果有其他线程在此改变了集合的结构，那也是在副本上的改变，而不是影响到原集合，迭代器还是照常遍历，**遍历完之后，改变原引用指向副本**。

总的一句话就是**如果在次包下的类进行增加删除，就会出现一个副本**。所以能防止fail-fast，这种机制并不会出错，所以我们叫这种现象为fail-safe。



下面是CopyOnWriteArrayList的add方法源码：

```
1.  public boolean add(E e) {  

2.      final ReentrantLock lock = this.lock;  

3.      lock.lock();  

4.      try {  

5.          Object[] elements = getArray();  

6.          int len = elements.length;  

7.          // 这里对原有的elementData进行了复制  

8.         Object[] newElements = Arrays.copyOf(elements, len + 1);  

9.          newElements[len] = e;  

10.         // 这里将复制的副本覆盖原有的elementData  

11.         setArray(newElements);  

12.         return true;  

13.     } finally {  

14.         lock.unlock();  

15.     }  

16. }  
```



**Q2：vector也是线程安全的，为什么是fail-fast呢？**

并不是说线程安全的集合就不会报fail-fast，而是报fail-safe，**出现的fail-fast的根本是底层对着真正的引用进行操作，所以才会发生异常。**



