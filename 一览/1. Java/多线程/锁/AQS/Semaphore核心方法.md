Semaphore是一种计数信号量，用于管理一组资源，内部是基于AQS的共享模式。它相当于给线程规定一个量从而控制允许活动的线程数。Semaphore更像一个许可证管理器。

就好像我们小区的停车场，停车位肯定是有限的，在停车位满了的情况下只能等待别人把车子开走才能停进来，那么这个场景用Semaphore是可以实现的。



简单的示例代码如下：

```
1.  public class SemaphoreTest {  

2.      private Semaphore semaphore = new Semaphore(3,true);  

3.      private ThreadPoolExecutor threadPool =  

4.          new ThreadPoolExecutor(100, 1000, 1, TimeUnit.SECONDS, new LinkedBlockingQueue<>(1000));  

5.      public static void main(String[] args) {  

6.          new SemaphoreTest().doPark();  

7.      }  

8.      private void doPark() {  

9.          for (int i = 0; i < 100; i++) {  

10.             threadPool.execute(() -> {  

11.                 String threadName = Thread.currentThread().getName();  

12.                 if("pool-1-thread-99".equals(threadName)){  

13.                     System.out.println("semaphore.getQueueLength() = " + semaphore.getQueueLength());  

14.                     System.out.println("semaphore.hasQueuedThreads() = " + semaphore.hasQueuedThreads());  

15.                     System.out.println("semaphore.availablePermits() = " + semaphore.availablePermits());  

16.                     System.out.println("semaphore.drainPermits() = " + semaphore.drainPermits());  

17.                 }  

18.                 try {  

19.                     System.out.println(threadName + "：我在准备停车");  

20.                     semaphore.acquire(1);  

21. //                    semaphore.acquire();  

22. //                    semaphore.acquireUninterruptibly(1);  

23. //                    semaphore.tryAcquire();  

24. //                    semaphore.tryAcquire(1);  

25. //                    semaphore.tryAcquire(2,TimeUnit.SECONDS);  

26.                 } catch (InterruptedException e) {  

27.                     e.printStackTrace();  

28.                 }  

29.                 System.out.println(threadName + "：我停进来了");  

30.                 System.out.println("");  

31.                 semaphore.release();  

32.                 System.out.println(threadName + "：我开走了");  

33.             });  

34.         }  

35.         threadPool.shutdown();  

36.         while (true) {  

37.             if (threadPool.isTerminated()) {  

38.                 System.out.println("线程池关闭！");  

39.                 break;  

40.             }  

41.         }  

42.     }  

43. }  
```

在这里一个停车位就是一个许可证，Semaphore是门口看车库的管理员。只有其他车子（线程）开走了（relase）,等待的车子（线程）才可以获取许可证进来（acquire）。



我们来看一下Semaphore对外开放的api

```
acquire()：阻塞等待获取一个许可证，可中断

acquire(int permits)：阻塞获取permits个许可证，可中断

acquireUninterruptibly()：阻塞等待获取一个许可证，不可中断

acquireUninterruptibly(int permits)：阻塞等待获取permits个许可证，不可中断

tryAcquire()：不阻塞等待获取一个许可证，立即返回

tryAcquire(int permits)：不阻塞等待获取permits个许可证，立即返回

tryAcquire(int permits, long timeout, TimeUnit unit)：不阻塞等待获取permits个许可证，有超时条件

availablePermits()：剩余的可用许可证数量

drainPermits()：立即获取所有的可用许可证，并返回数量

getQueueLength()：获取正在等待许可证的线程数量

hasQueuedThreads：判断有没有线程在等待许可证

release()：释放一个许可证

release(int permits)：释放permits个许可证
```



看完api方法后，来看一下重要方法的源码

#### 1. acquire()

```
  
2.  public void acquire() throws InterruptedException {  

3.      sync.acquireSharedInterruptibly(1);  

4.  }  


5.  public final void acquireSharedInterruptibly(int arg)  

6.          throws InterruptedException {  

7.      if (Thread.interrupted())  

8.          throw new InterruptedException();  

9.      if (tryAcquireShared(arg) < 0)  

10.         doAcquireSharedInterruptibly(arg);  

11. }  
```



##### 1.1 tryAcquireShared(int acquires)

看一下Semphore重写的的tryAcquireShared方法

分fair和nonfair两种

在nonfair中

```
1.  protected int tryAcquireShared(int acquires) {  

2.      return nonfairTryAcquireShared(acquires);  

3.  }  


4.  final int nonfairTryAcquireShared(int acquires) {  

5.      for (;;) {  

6.          int available = getState();  

7.          int remaining = available - acquires;  

8.          if (remaining < 0 ||  

9.              compareAndSetState(available, remaining))  

10.             return remaining;  

11.     }  

12. }  
```

可以看出就是看看剩余的许可证数是不是大于我需要的许可证数，如果大于的就通过CAS将剩余的许可证数修改成减少后的数量，然后返回获取资源成功



在fair中

```
1.  protected int tryAcquireShared(int acquires) {  

2.      for (;;) {  

3.          if (hasQueuedPredecessors())  

4.              return -1;  

5.          int available = getState();  

6.          int remaining = available - acquires;  

7.          if (remaining < 0 ||  

8.              compareAndSetState(available, remaining))  

9.              return remaining;  

10.     }  

11. }  
```

可以看出跟nonfair相比就多了一个hasQueuedPredecessors判断操作，fair就话必须要求当前线程节点排着队来获取资源。



**如果获取不到资源当前线程怎么办呢？**

doAcquireSharedInterruptibly()方法会将当前线程阻塞等待唤醒

```
1.  private void doAcquireSharedInterruptibly(int arg)  

2.      throws InterruptedException {  

3.      final Node node = addWaiter(Node.SHARED);  

4.      boolean failed = true;  

5.      try {  

6.          for (;;) {  

7.              final Node p = node.predecessor();  

8.              if (p == head) {  

9.                  int r = tryAcquireShared(arg);  

10.                 if (r >= 0) {  

11.                     setHeadAndPropagate(node, r);  

12.                     p.next = null; // help GC  

13.                     failed = false;  

14.                     return;  

15.                 }  

16.             }  

17.             if (shouldParkAfterFailedAcquire(p, node) &&  

18.                 parkAndCheckInterrupt())  

19.                 throw new InterruptedException();  

20.         }  

21.     } finally {  

22.         if (failed)  

23.             cancelAcquire(node);  

24.     }  

25. }  
```

AQS中熟悉的操作，就不细讲了，对AQS独占式感兴趣的可以看ReentrantLock源码分析，对AQS共享式感兴趣的可以看CountDownLatch源码分析。



acquire和acquireUninterruptibly的区别就是对interrupt事件的反应

acquire()如下

```
1.  if (shouldParkAfterFailedAcquire(p, node) &&  

2.      parkAndCheckInterrupt())  

3.      throw new InterruptedException();  
```

会抛出InterruptedException



acquireUninterruptibly()如下

```
1.  if (shouldParkAfterFailedAcquire(p, node) &&  

2.      parkAndCheckInterrupt())  

3.      interrupted = true;  
```

只是会记录一下interrupted，不会做出任何处理，该阻塞的还是阻塞



来看一下tryAcquire()

```
1.  public boolean tryAcquire() {  

2.      return sync.nonfairTryAcquireShared(1) >= 0;  

3.  }  
```

会直接返回能不能立即获取到资源



#### 2. release()

release会释放许可证，当然释放了许可证意味着有了资源去争夺，所以会唤醒所有被park的线程

```
1.  public void release() {  

2.      sync.releaseShared(1);  

3.  }  


4.  public final boolean releaseShared(int arg) {  

5.      if (tryReleaseShared(arg)) {  

6.          doReleaseShared();  

7.          return true;  

8.      }  

9.      return false;  

10. }  
```



根据上面的思路，先释放许可证，看一下Semaphore重写的tryReleaseShared方法

```
1.  protected final boolean tryReleaseShared(int releases) {  

2.      for (;;) {  

3.          int current = getState();  

4.          int next = current + releases;  

5.          if (next < current) // overflow  

6.              throw new Error("Maximum permit count exceeded");  

7.          if (compareAndSetState(current, next))  

8.              return true;  

9.      }  

10. }  
```

就是增加许可证数量，加了边界值判断



然后unpark同步队列中阻塞的线程节点，看AQS中的doReleaseShared方法

```
1.  private void doReleaseShared() {  

2.      for (;;) {  

3.          Node h = head;  

4.          if (h != null && h != tail) {  

5.              int ws = h.waitStatus;  

6.              if (ws == Node.SIGNAL) {  

7.                  if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))  

8.                      continue;            // loop to recheck cases  

9.                  unparkSuccessor(h);  

10.             }  

11.             else if (ws == 0 &&  

12.                      !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))  

13.                 continue;                // loop on failed CAS  

14.         }  

15.         if (h == head)                   // loop if head changed  

16.             break;  

17.     }  

18. }  
```

非常熟悉的大兄弟CyclicBarrier的操作，通知队列中所有的后继节点去获取资源，这也是AQS共享式的体现。



总结：

**Semaphore底层还是基于AQS共享式的，它通过一种许可证的方式进行资源的分配，允许多个线程在获取到资源的情况下运行，对于资源的获取和释放也可以是批量的。**