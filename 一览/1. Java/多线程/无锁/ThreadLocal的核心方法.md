#### 1. set方法

源码如下

```
1.  public void set(T value) {  

2.      Thread t = Thread.currentThread();  

3.      ThreadLocalMap map = getMap(t);  

4.      if (map != null)  

5.          map.set(this, value);  

6.      else  

7.          createMap(t, value);  

8.  }  
```



#### 2. getMap()代码

```
1.  ThreadLocalMap getMap(Thread t) {  

2.      return t.threadLocals;  

3.  }  
```



t.threadLocals是什么呢？

```
1.  class Thread implements Runnable {  

2.      //...  

3.      ThreadLocal.ThreadLocalMap threadLocals = null;  

4.      //...  

5.  }  
```

可以看到threadLocals是Thread里面的一个变量，它的类型是ThreadLocal.ThreadLocalMap，这个ThreadLocalMap下面再说，可以看到这里threadLocals赋值null

那在哪里初始化的？



还是回到set()方法，在map == null的时候，createMap(t, value)方法对threadLocals进行初始化

createMap代码如下

```
1.  void createMap(Thread t, T firstValue) {  

2.      t.threadLocals = new ThreadLocalMap(this, firstValue);  

3.  }  
```



现在set()方法的流程就很清晰了

1. 寻找当前线程的threadLocals，有的话直接put值，没有的话新建一个
2. put值的时候，是将当前的ThreadLocal作为ThreadLocalMap的key，set方法的参数value作为ThreadLocalMap的value。



看到这儿其实已经知道ThreadLocal怎么实现线程安全的了，对每一个线程都维护一个threadLocals变量，然后将当前的（threadLocal，value）插入threadLocals中。



ThreadLocal和Thread的关系如下图：

![img](http://pcc.huitogo.club/b66dd36bddad099dd26ba8f933c31306)



#### 3. get方法

get方法源码如下

```
1.  public T get() {  

2.      Thread t = Thread.currentThread();  

3.      ThreadLocalMap map = getMap(t);  

4.      if (map != null) {  

5.          ThreadLocalMap.Entry e = map.getEntry(this);  

6.          if (e != null) {  

7.              @SuppressWarnings("unchecked")  

8.              T result = (T)e.value;  

9.              return result;  

10.         }  

11.     }  

12.     return setInitialValue();  

13. }  
```



可以看到获取到了当前线程的threadLocals里面key为this.threadLocal的value值，如果当前线程threadLocals == null的话，初始化操作如下：

```
1.  private T setInitialValue() {  

2.      T value = initialValue();  

3.      Thread t = Thread.currentThread();  

4.      ThreadLocalMap map = getMap(t);  

5.      if (map != null)  

6.          map.set(this, value);  

7.      else  

8.          createMap(t, value);  

9.      return value;  

10. }  
```

主要也就set/get方法了。



#### 4. ThreadLocalMap是什么？

ThreadLocalMap从字面上就可以看出这是一个保存ThreadLocal对象的map(其实是以它为Key)

##### 4.1 Entry

跟所有Map集合一样，ThreadLocalMap维护的也是一个Entry数组，初始化大小为16

但是ThreadLocalMap的Entry其实是包装后的ThreadLocal



Entry代码如下：

```
1.  static class Entry extends WeakReference<ThreadLocal<?>> {  

2.      Object value;  

3.     // 这里可以看到Entry的key是一个弱引用

4.      Entry(ThreadLocal<?> k, Object v) {  

5.          super(k);  

6.          value = v;  

7.      }  

8.  }  
```

可以看到它就是个扩展了弱引用类型的ThreadLocal对象，并且它里面的“key”就是个弱引用的ThreadLocal。



##### 4.2 构造函数

ThreadLocal的一个构造函数如下：

```
1.  ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {  

2.      table = new Entry[INITIAL_CAPACITY];  

3.      int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);

4.      table[i] = new Entry(firstKey, firstValue);  

5.      size = 1; 

6.  // 设置扩容阀值 

7.      setThreshold(INITIAL_CAPACITY);  

8.  }  
```

可以看到它内部维护的Entry数组，并且初始化大小为16，在size > (2/3)length的时候会扩容。



这里我们可以看到ThreadLocal是怎么定位一个key在Entry数组中的位置

```
firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1) 
```

使用一个 static 的原子属性 AtomicInteger nextHashCode，通过每次增加 HASH_INCREMENT = 0x61c88647 ，然后 & (INITIAL_CAPACITY - 1) 取得在数组 private Entry[] table 中的索引。

基本可以将ThreadLocalMap作为一个简单的Map集合看待。