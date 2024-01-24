**1. ThreadLocal的内存回收**

**1.1 Thread的回收**

threadLocals作为Thread的一个变量，在Thread执行完毕后就会自动回收。



Thread的exit方法如下

```
1.  private void exit() {  

2.       if (group != null) {  

3.           group.threadTerminated(this);  

4.           group = null;  

5.       }  

6.       target = null;  

7.  // 这里线程退出时将threadLocals 置为null

8.       threadLocals = null;  

9.       inheritableThreadLocals = null;  

10.      inheritedAccessControlContext = null;  

11.      blocker = null;  

12.      uncaughtExceptionHandler = null;  

13.  } 
```



**1.2 ThreadLocalMap的回收**

在一个Thread运行时间长的情况下，ThreadLocalMap也会根据情况在线程的生命期内回收ThreadLocalMap 的内存了，不然的话，Entry对象越多，那么ThreadLocalMap就会越来越大，占用的内存就会越来越多，所以对于已经不需要了的线程局部变量，就应该清理掉其对应的Entry对象。



**怎么回收的呢？**

还记得ThreadLocalMap.Entry的key是一个WeakReference对象吗？其实这也是ThreadLocalMap的一个细节，因为**当一个对象仅仅被weak reference指向, 而没有任何其他strong reference指向的时候, 如果GC运行, 那么这个对象就会被回收。**

所以在jvm发生GC的时候，ThreadLocalMap.Entry里面的key会被回收，也就是key被置为null，但是key所在Entry还在存在的，那Entry什么时候回收？



ThreadLocalMap中在执行get、set、remove方法的时候都会对key为null的Entry进行处理

比如get中会清除过期对象

```
1.  private Entry getEntryAfterMiss(ThreadLocal key, int i, Entry e) {  

2.      Entry[] tab = table;  

3.      int len = tab.length;  

4.      while (e != null) {  

5.          ThreadLocal k = e.get();  

6.          if (k == key)  

7.              return e;  

8.         if (k == null) 

9.             expungeStaleEntry(i);  

10.         else  

11.             i = nextIndex(i, len);  

12.         e = tab[i];  

13.     }  

14.     return null;  

15. } 
```



在set中会替换过期对象

```
1.  private void set(ThreadLocal key, Object value) {  

2.       Entry[] tab = table;  

3.       int len = tab.length;  

4.       int i = key.threadLocalHashCode & (len-1);  

5.       for (Entry e = tab[i];  

6.            e != null;  

7.            e = tab[i = nextIndex(i, len)]) {  

8.           ThreadLocal k = e.get();  

9.           if (k == key) {  

10.              e.value = value;  

11.              return;  

12.          }  

13.          if (k == null) { 

14.              replaceStaleEntry(key, value, i);  

15.              return; 

16.          }  

17.      }  

18.      tab[i] = new Entry(key, value);  

19.      int sz = ++size;  

20.      if (!cleanSomeSlots(i, sz) && sz >= threshold)  

21.          rehash();  

22.  } 
```



除此之外当数组长度 > 2/3 最大长度时发生扩容之前会尝试先进行回收

```
1.  if (!cleanSomeSlots(i, sz) && sz >= threshold)  

2.      rehash();  
```



cleanSomeSlots回收方法源码如下：

```
1.  private boolean cleanSomeSlots(int i, int n) {  

2.      boolean removed = false;  

3.      Entry[] tab = table;  

4.      int len = tab.length;  

5.      do {  

6.          i = nextIndex(i, len);  

7.          Entry e = tab[i];  

8.          if (e != null && e.get() == null) {  

9.              n = len;  

10.             removed = true;  

11.             i = expungeStaleEntry(i);  

12.         }  

13.     } while ( (n >>>= 1) != 0);  

14.     return removed;  

15. }  
```



**2. ThreadLocal会引起OOM吗？**

理论上是不会的，从上面ThreadLocal的内存回收可以看的出来，无论在线程执行后还是执行中都是会进行回收保障的，但是！如果这个线程一直不退出呢？

没错，就是使用线程池的情况，比如固定线程池，线程一直存活，那么可想而知

如果处理一次业务的时候存放到ThreadLocalMap中一个大对象，处理另一个业务的时候，又一个线程存放到ThreadLocalMap中一个大对象，但是这个线程由于是线程池创建的他会一直存在，不会被销毁，这样的话，以前执行业务的时候存放到ThreadLocalMap中的对象可能不会被再次使用，但是由于线程不会被关闭，因此无法释放Thread中的ThreadLocalMap对象，造成内存溢出。

因此在线程池和ThreadLocal的使用下，是有可能出现OOM溢出的。



但是ThreadLocal出现OOM的原因到底是什么呢？

我们知道ThreadLocalMap threadlocals是Thread的全局变量，**它的生命周期和Thread是一样的**，虽然在ThreadLocalMap.Entry中的key是一个WeakReference，但是也只是在GC的时候清除这个key，然后key所在的Entry对象只会在当前ThreadLocal发生get、set方法后清除，所以如果当前**ThreadLocal在多线程set完大对象之后没有发生get/set行为**，那么这些key为null的Entry就一直存在！



ThreadLoca测试OOM溢出的代码如下：

```
1.  public class ThreadLocalOOM {  

2.      public static void main(String[] args) {  

3.          ExecutorService executorService = Executors.newFixedThreadPool(500);  

4.          ThreadLocal<List<User>> threalLocal = new ThreadLocal<>();  

5.          try {  

6.              for(int i =0; i < 100; i++) {  

7.                  executorService.execute(() ->{  

8.                      threalLocal.set(new ThreadLocalOOM().addBigList());  

9.                      System.out.println(Thread.currentThread().getName() + "has executed!");  

10.                     threalLocal.remove();  

11.                 });  

12.             }  

13.             Thread.sleep(1000L);  

14.         } catch (Exception e) {  

15.             e.printStackTrace();  

16.         }  

17.     }  


18.     public List<User> addBigList(){  

19.         List<User> list = new ArrayList<>(10000);  

20.         for(int j = 0; j < 10000; j++) {  

21.             list.add(new User(Thread.currentThread().getName(),"this is a test User" + j, 18884848, 25545454));  

22.         }  

23.         return list;  

24.     }  

25. }  


26. class User {  

27.     private String name;  

28.     private String description;  

29.     private int age;  

30.     private int sex;  

31.     public User(String name, String description, int age, int sex) {  

32.         super();  

33.         this.name = name;  

34.         this.description = description;  

35.         this.age = age;  

36.         this.sex = sex;  

37.     }  

38. }  
```



这里将Eclipse的Run configuration的jvm参数为-Xms100M -Xmx100M

报错的话是系统没办法在堆内存里面再给ArrayList分配内存了，因为每一个ArrayList都是大对象，然后放到ThreadLocal中又没有清除，所以OOM了。。。



**解决办法就是在使用完ThreadLocal后手动触发回收，也就是ThreadLocal.remove()**

当然这也是建立在业务的基础上的，如果希望ThreadLocal中set的值生命周期变长的话就不能随便remove了，但是要记住，**不要给ThreadLocal里面set大对象**！