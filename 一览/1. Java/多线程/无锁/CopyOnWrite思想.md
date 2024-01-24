这里基于CopyOnWriteArrayList和CopyOnWriteArraySet讲解一下CopyOnWrite思想

可以类比一下ReentrantReadWriteLock的思想，读写互斥、写写互斥、写读互斥，在读读的时候可以并发进行。然而CopyOnWrite思想更高级，在读读的时候根本不加锁，**只在写写的时候进行加锁处理**。

CopyOnWrite思想就是说在计算机中，如果你想要对一块内存进行修改时，我们不在原有内存块中进行写操作，而是将内存拷贝一份，在新的内存中进行写操作，写完之后呢，就将指向原来内存指针指向新的内存，原来的内存就可以被回收掉了。



可以简单看一下CopyOnWriteArrayList的add源码

```
1.  public boolean add(E e) {  

2.      final ReentrantLock lock = this.lock;  

3.      lock.lock();  

4.      try {  

5.          Object[] elements = getArray();  

6.          int len = elements.length;  

7.          Object[] newElements = Arrays.copyOf(elements, len + 1);  

8.          newElements[len] = e;  

9.          setArray(newElements);  

10.         return true;  

11.     } finally {  

12.         lock.unlock();  

13.     }  

14. }  
```