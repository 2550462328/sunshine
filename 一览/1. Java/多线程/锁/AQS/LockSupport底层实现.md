之前在看ReentrantLock.lock和ReentrantLock.unlock方法中可以看到最后阻塞线程和激活线程使用的是LockSupport.park()和LockSupport.unpark()方法，那么LockSupport是什么？

我认为LockSupport相当于Lock的一个工具类，主要用途就是**实现对线程的阻塞和解除对线程的阻塞**

park()和unpark()的原理是基于“许可证”（一种信号量机制），park的时候消耗一个“许可证”，unpark()的时候会提供一个“许可证”（注意多次不会叠加），当许可证为0的时候，线程就会阻塞了，等待发放“许可证”。



#### 1. park方法

##### 1.1 park()

```
3.   public static void park() {  

4.       UNSAFE.park(false, 0L);  

5.   }  

6.  //unsafe.class中  

7.  public native void park(boolean var1, long var2);  
```

调用jvm的本地方法将当前线程阻塞。



##### 1.2 park(Object blocker)

```
2.  public static void park(Object blocker) {  

3.      Thread t = Thread.currentThread();  

4.      setBlocker(t, blocker);  

5.      UNSAFE.park(false, 0L);  

6.      setBlocker(t, null);  

7.  }  

8.  private static void setBlocker(Thread t, Object arg) {  

9.      UNSAFE.putObject(t, parkBlockerOffset, arg);  

10. }  
```

这里在线程阻塞前和线程阻塞后都会setBlocker()，但是setBlocker有什么用？为什么设置两次？

这里第一个setBlocker就是设置当前Thread对象中的parkBlocker变量，**用来记录线程被阻塞时被谁阻塞的,用于线程监控和分析工具来定位原因的**。

第二个setBlocker是进行还原操作，防止另一个Thread获取到了错误了parkBlocker。



获取这个blocker采用的是内存偏移量方式，查找parkBlocker字段相对于Thread的内存偏移量，然后根据这个内存偏移量去查找变量值

```
1.  public static Object getBlocker(Thread t) {  

2.      if (t == null)  

3.          throw new NullPointerException();  

4.      return UNSAFE.getObjectVolatile(t, parkBlockerOffset);  

5.  }  

6.  parkBlockerOffset = UNSAFE.objectFieldOffset(tk.getDeclaredField("parkBlocker"));  
```



**为什么要基于内存的方式去设置/获取blocker**？直接写个方法不就行了？

parkBlocker就是在线程处于阻塞的情况下才被赋值。线程都已经被阻塞了,如果不通过这种内存的方法,而是直接调用线程内的方法,线程是不会回应调用的。



关于blocker做的小案例，获取blocker的变换情况

```
1.  public static void main(String[] args) throws InterruptedException {  

2.      Thread thread = Thread.currentThread();  

3.      Thread thread1 = new Thread(() -> {  

4.          try {  

5.              TimeUnit.SECONDS.sleep(1);  

6.          } catch (InterruptedException e) {  

7.          }  

8.          System.out.println(thread.getName() + "before block : block info is " + LockSupport.getBlocker(thread));  

9.          LockSupport.unpark(thread);  

10.     });  

11.     thread1.start();  

12.     LockSupport.park(new String("aa"));  

13.     System.out.println(thread.getName() + "after block : block info is " + LockSupport.getBlocker(thread));  

14. }  
```



输出

```
1.  mainbefore block : block info is blocker  

2.  mainafter block : block info is null  
```



##### 1.3 parkNanos(Object blocker, long nanos)

```
1.  public static void parkNanos(Object blocker, long nanos) {  

2.      if (nanos > 0) {  

3.          Thread t = Thread.currentThread();  

4.          setBlocker(t, blocker);  

5.          UNSAFE.park(false, nanos);  

6.          setBlocker(t, null);  

7.      }  

8.  }  
```

和park(Objectblocker)的区别就是多个限时操作，在规定时间内没有获取到“许可证”会直接返回。



park在下面三种情况会直接返回

- 调用线程被中断,则park方法会返回
- unpark方法先于park调用会直接返回
- 其他某个线程将当前线程作为目标调用 unpark
- park方法还可以在其他任何时间"毫无理由"地返回,因此**通常必须在重新检查返回条件的循环里调用此方法**。从这个意义上说,park是“忙碌等待”的一种优化,它不会浪费这么多的时间进行自旋,但是必须将它与 unpark配对使用才更高效



#### 2. unpark

```
2.  public static void unpark(Thread thread) {  

3.      if (thread != null)  

4.          UNSAFE.unpark(thread);  

5.  }  

6.  //unsafe.class中  

7.  public native void unpark(Object var1);  
```

解除指定线程的阻塞，必须指定线程。



park()和unpark()不会有“Thread.suspend和Thread.resume所可能引发的死锁”问题,由于许可证的存在，调用park 的线程和另一个试图将其 unpark 的线程之间的竞争将保持活性。



#### 3. park/unpark和wait/nofity的区别

首先对比一下调用park和wait后通过jstack打印出来的堆栈情况

![img](http://pcc.huitogo.club/f52b1a35236229069c79db578b44eff2)

![img](http://pcc.huitogo.club/6a43d4c94b4b97c81f9fde8fd16b807c)



两者区别如下：

1. LockSuport主要是针对Thread进行阻塞处理,可以指定阻塞队列的目标对象,每次可以指定具体的线程唤醒。Object.wait()是以对象为纬度,阻塞当前的线程和唤醒单个(随机)或者所有线程。
2. Thead在调用wait之前，当前线程必须先获得该对象的监视器(synchronized)，被唤醒之后需要重新获取到监视器才能继续执行.而LockSupport可以随意进行park或者unpark。
3. park时遇到interrupt会直接返回继续执行，而wait的时候遇到interrupt会抛出异常。