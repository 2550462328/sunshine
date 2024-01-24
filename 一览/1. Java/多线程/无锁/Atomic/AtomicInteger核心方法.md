下面是AtomicInteger的一些重要源码

### 1. compareAndSet

这个方法是AtomicInteger的灵魂，几乎所有方法都是这个原理，都需要调用unSafe的compareAndSwapInt

这里参数一个是期待值，一个是修改值，如果期待值不等于原值，不做修改，允许再次执行。



AtomicInteger的源码

```
1.  public final boolean compareAndSet(int expect, int update) {  

2.      return unsafe.compareAndSwapInt(this, valueOffset, expect, update);  

3.  }  
```



Unsafe类的源码

```
1.  public final native boolean compareAndSwapInt(Object o, long offset,  

2.                                                int expected,  

3.                                                int x); 
```

这里compareAndSwapInt方法

第一个参数是目标对象，也就是内存中的目标对象

第二个参数是目前值相对于内存中目标对象的偏移量

第三个参数就是期待值，这个就是乐观锁的基础，只有当期待值跟原值一样的时候才可以修改

第四个参数就是即将替换的值



### 2. incrementAndGet

将value + 1并且返回value +1

AtomicInteger的源码

```
1.  public final int getAndIncrement() {  

2.      return unsafe.getAndAddInt(this, valueOffset, 1);  

3.  }  
```



Unsafe类的源码

```
1.  public final int getAndAddInt(Object o, long offset, int delta) {  

2.      int v;  

3.      do {  

4.          v = getIntVolatile(o, offset);  

5.      } while (!compareAndSwapInt(o, offset, v, v + delta));  

6.      return v;  

7.  }  
```

可以看到这边是做一个死循环，直到成功修改到了值为止。



```
public native int   getIntVolatile(Object o, long offset); 
```

这个本地方法是获取某个对象在内存中在偏移量offset处的值。

还有getAndSet()和getAndAdd()等等都类似于这个



### 3. lazySet

延迟设置newValue，跟set(newValue)类似

经常在维护一个非阻塞的数据结构时，内部显然常常会有volatile成员，而对于通信队列这样的对性能要求极端的数据结构，JMM（java memory model）要维持volatile的内存可见性，在对这样的成员写操作时，会有频繁的cache的读写（save和load？）到主内存。这样的时间耗损对性能而言，其实也十分重要。

lazySet()可以让一个线程在没有其他线程volatile写或者synchronized动作发生前，本地缓存的写操作不必刷回主内存。



AtomicInteger的源码

```
1.  public final void lazySet(int newValue) {  

2.      unsafe.putOrderedInt(this, valueOffset, newValue);  

3.  } 
```



Unsafe类的源码

```
public native void  putOrderedInt(Object o, long offset, int x); 
```



### 4. accumulateAndGet

这是jdk1.8引入函数式编程后引入的方法

意思就是给个数x，然后跟atomicInteger的value在accumulatorFunction的函数计算后返回计算后的值



AtomicInteger的源码

```
1.  public final int accumulateAndGet(int x,  IntBinaryOperator accumulatorFunction) {  

3.      int prev, next;  

4.      do {  

5.          prev = get();  

6.          next = accumulatorFunction.applyAsInt(prev, x);  

7.      } while (!compareAndSet(prev, next));  

8.      return next;  

9.  } 
```



实例运用中需要结合函数式用

```
1.  AtomicInteger atomic = new AtomicInteger(2);  

2.  System.out.println(atomic.accumulateAndGet(5, Math::max)); // print 5  
```

还有getAndAccumulate方法就类似这个了，一个是返回prev，一个返回next



### 5. updateAndGet

这是也是jdk1.8引入函数式编程后引入的方法

使用效果和上面类似



AtomicInteger的源码

```
1.  public final int updateAndGet(IntUnaryOperator updateFunction) {  

2.      int prev, next;  

3.      do {  

4.          prev = get();  

5.          next = updateFunction.applyAsInt(prev);  

6.      } while (!compareAndSet(prev, next));  

7.      return next;  

8.  }  
```



实例运用中需要结合函数式用

```
1.  AtomicInteger atomic = new AtomicInteger(2);  

2.  System.out.println(atomic.updateAndGet(x -> Math.max(x,4))); // print 5  
```

AtomicInteger的主要方法就是上面这些，[Unsafe的实现源码](https://blog.csdn.net/qqqqq1993qqqqq/article/details/75211993)（native method）是用C写的，看的懂C得可以去看这个



**compareAndSweepInt中的第二个参数offset怎么算的？**

在AtomicInteger的源码中是这么做的

```
1.  private static final long valueOffset;  

2.  private volatile int value;  

3.  static {  

4.      try {  

5.          valueOffset = unsafe.objectFieldOffset (AtomicInteger.class.getDeclaredField("value"));  

7.      } catch (Exception ex) { throw new Error(ex); }  

8.  }
```

通过反射获取属性在内存中相对于对象的偏移量，主要还是调用了unsafe.objectFieldOffset(Field f)方法。