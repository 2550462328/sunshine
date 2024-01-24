### 1. ReferenceQueue介绍

**ReferenceQueue内部实现实际上是一个栈**



下面是ReferenceQueue做数据监控的实例：

```
1.  private static ReferenceQueue<byte[]> rq = new ReferenceQueue<>();  

2.  private static int _1M = 1024 * 1024;  


3.  public static void main(String[] args) {  

4.      Map<WeakReference<byte[]>, Object> map = new HashMap<>();   

5.      Object value = new Object();  

6.      Thread thread = new Thread(MyArrayDeque :: run);  

7.      thread.setDaemon(true);  

8.      thread.start();  

9.      // 放入数据  

10.     for(int i = 0; i < 100; i++) {  

11.         byte[] bytes = new byte[_1M];  

12.         WeakReference<byte[]> weakReference = new WeakReference<>(bytes, rq);  

13.         map.put(weakReference, value);  

14.     }  

15.     System.out.println("map.size = " + map.size());  

16.     // 获取存活数据  

17.     int aliveNum = 0;  

18.     for(Entry<WeakReference<byte[]>, Object> entry : map.entrySet()) {  

19.         if(entry != null) {  

20.             if(entry.getKey().get() != null) {  

21.                 aliveNum++;  

22.             }  

23.         }  

24.     }  

25.     System.out.println("100个对象存活的对象个数 = " + aliveNum);  

26. }  


27. // 监控线程  

28. public static void run() {  

29.     int n=0;  

30.     WeakReference k;  

31.     try {  

32.         while((k = (WeakReference) rq.remove()) != null) {  

33.             System.out.println(++n + "回收了" + k);  

34.         }  

35.     } catch (InterruptedException e) {  

36.         e.printStackTrace();  

37.     }  

38. }  
```



从上面实例我们知道，在map中的WeakReference对象失效后（被系统回收），会放入referenceQueue中，我们可以监控refenceQueue（通过阻塞的方式获取数据）来得知回收情况。



### 2. 成员变量

```
3.  static ReferenceQueue<Object> NULL = new Null<>();  

4.  static ReferenceQueue<Object> ENQUEUED = new Null<>();  

5.  // ReferenceQueue采用单链表的方式存储Reference

6.  private volatile Reference<? extends T> head = null;

7.  // 队列长度，线程安全

8.  private long queueLength = 0;

9.  // 加lock 做同步对象

10. static private class Lock { };

11. private Lock lock = new Lock();
```



NULL对象

```
1.  private static class Null<S> extends ReferenceQueue<S> {  

2.      boolean enqueue(Reference<? extends S> r) {  

3.          return false;  

4.      }  

5.  }  
```

在ReferenceQueue中入队（enqueue）时，将新来的Reference对象的queue置为ENQUEUED，在ReferenceQueue中出队（poll或者remove时），将被移除的Reference对象的queue置为NULL



**Q1：这里为什么不直接new 两个ReferenceQueue，毕竟NULL只是简单继承了ReferenceQueue？**

主要是因为ReferenceQueue的enqueue方法是加lock的，且**只用来给Reference类调用**，这里是**怕有人在ReferenceQueue中对NULL和ENQUEUED误操作**，即调用了enqueue方法，所以生成一个子类继承ReferenceQueue类并且覆盖了enqueue方法，直接return false。



### 3. 主要方法

#### 3.1 入队

```
3.  // 这个方法仅会被Reference类调用  

4.  boolean enqueue(Reference<? extends T> r) {   

5.      synchronized (lock) {  

6.          // 检测从获取这个锁之后，该Reference没有入队，并且没有被移除  

7.          ReferenceQueue<?> queue = r.queue;  

8.          if ((queue == NULL) || (queue == ENQUEUED)) {  

9.              return false;  

10.         }  

11.         assert queue == this;  

12.         // 将reference的queue标记为ENQUEUED  

13.         r.queue = ENQUEUED;  

14.         // 将r设置为链表的头结点  

15.         r.next = (head == null) ? r : head;  

16.         head = r;  

17.         queueLength++;  

18.         // 如果r的FinalReference类型，则将FinalRef+1  

19.         if (r instanceof FinalReference) {  

20.             sun.misc.VM.addFinalRefCount(1);  

21.         }  

22.         lock.notifyAll();  

23.         return true;  

24.     }  

25. }  
```

这里是入队的方法，使用了lock对象锁进行同步，将传入的r添加到队列中，并重置头结点为传入的节点。



#### 3.2  出队

出队分两种，阻塞和非阻塞



**非阻塞**，直接弹出头节点

```
1.  public Reference<? extends T> poll() {  

2.      if (head == null)  

3.          return null;  

4.      synchronized (lock) {  

5.          return  reallyPoll();  

6.      }  

7.  }  


8.  private Reference<? extends T> reallyPoll() {       

9.      Reference<? extends T> r = head;  

10.     if (r != null) {  

11.         head = (r.next == r) ? null : r.next;  

13.         r.queue = NULL;  

14.         r.next = r;  

15.         queueLength--;  

16.         if (r instanceof FinalReference) {  

17.             sun.misc.VM.addFinalRefCount(-1);  

18.         }  

19.         return r;  

20.     }  

21.     return null;  

22. }  
```

注意：**这里弹出的是头节点，也就是最新的元素，所以ReferenceQueue类似于Stack（先进后出）**



**阻塞**，**阻塞到获取到一个Reference对象或者超时才会返回**

```
1.  /** 

2.    * 移除并返回队列首节点，此方法将阻塞到获取到一个Reference对象或者超时才会返回 

3.    * timeout时间的单位是毫秒 

4.    */  

5.  public Reference<? extends T> remove(long timeout)  

6.      throws IllegalArgumentException, InterruptedException{  

7.      if (timeout < 0) {  

8.          throw new IllegalArgumentException("Negative timeout value");  

9.      }  

10.     synchronized (lock) {  

11.         Reference<? extends T> r = reallyPoll();  

12.         if (r != null) return r;  

13.         long start = (timeout == 0) ? 0 : System.nanoTime();  

14.         // 死循环，直到取到数据或者超时  

15.         for (;;) {  

16.            lock.wait(timeout);  

17.             r = reallyPoll();  

18.             if (r != null) return r;  

19.             if (timeout != 0) {  

20.                 // System.nanoTime方法返回的是纳秒，1毫秒=1纳秒*1000*1000  

21.                 long end = System.nanoTime();  

22.                 timeout -= (end - start) / 1000_000;  

23.                 if (timeout <= 0) return null;  

24.                 start = end;  

25.             }  

26.         }  

27.     }  

28. }  
```



remove方法和poll的区别就是它会**阻塞到超时或者取到一个Reference对象才会返回**。



**Q1：在remove方法的阻塞时间内，lock锁是一直被占用的，那么其他需要用到lock锁的方法，比如enqueue可以使用吗？**

答案是可以的，正如我们文章开头的例子里面，一边阻塞的从ReferenceQueue中获取数据，一边向HashMap中添加WeakReference对象，也就会触发WeakReference的enqueue方法。原因就是**虚拟机会通过其他方式将Reference对象塞进去**。（具体什么方式，呃呃呃）



4. ### ReferenceQueue的应用场景

ReferenceQueue一般用来与SoftReference、WeakReference或者PhantomReference配合使用，将需要关注的引用对象注册到引用队列后，便可以通过监控该队列来判断关注的对象是否被回收，从而执行相应的方法



主要使用场景：

1. 使用引用队列进行数据监控，类似前面栗子的用法。
2. 队列监控的反向操作，即意味着一个数据变化了，可以通过Reference对象反向拿到相关的数据，从而进行后续的处理。