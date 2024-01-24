ReentrantLock中对共享资源的独占主要是通过AQS中的成员变量state来控制，通过CAS操作state值来实现加锁和释放锁。同时那些没有获取到锁的线程就会被放到AQS中维护的一个FIFO双向队列中，将它们阻塞起来，当state值被修改成0的时候（有线程释放锁了），这些线程会被唤醒去尝试修改state的值（获取锁）。



整个过程如下图

![img](http://pcc.huitogo.club/88b658ccb23164875654dc6252f917e1)



#### 1. lock方法

```
2.  public void lock() {  

3.      sync.lock();  

4.  }  
```



这里sync的设计使用了模板类，如下

![img](http://pcc.huitogo.club/93b5e6c23745049e2af15c5cb5a7d29b)

当ReentrantLock的构造是公平锁的时候，会调用FairSync.lock()，非公平锁的时候调用NonfairSync.lock()，所有释放锁和尝试获取锁也类似。



我们先看看不公平锁，到底怎么不公平了？

NonfairSync.lock()

```
1.  final void lock() {  

2.      if (compareAndSetState(0, 1))  

3.          setExclusiveOwnerThread(Thread.currentThread());  

4.      else  

5.          acquire(1);  

6.  }  
```

这里就是CAS设置state的值，设置成功就留下名字证明当前资源被我占了（独占），设置设置失败的话会再给机会或者加入队列进行阻塞



acquire方法

```
1.  public final void acquire(int arg) {  

2.      if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))  

4.          selfInterrupt();  

5.  } 
```

tryAcquire方法尝试获取锁，如果成功就返回，如果不成功，则把当前线程和等待状态信息构建成一个Node节点，并将节点放入同步队列的尾部。然后为同步队列中的当前节点循环等待获取锁，直到成功。



重点看下tryAcquire方法

```
1.  protected final boolean tryAcquire(int acquires) {  

2.      return nonfairTryAcquire(acquires);  

3.  }  

4.  final boolean nonfairTryAcquire(int acquires) {  

5.      final Thread current = Thread.currentThread();  

6.      int c = getState();  

7.      if (c == 0) {  

8.          if (compareAndSetState(0, acquires)) {  

9.              setExclusiveOwnerThread(current);  

10.             return true;  

11.         }  

12.     }  

13.     else if (current == getExclusiveOwnerThread()) {  

14.         int nextc = c + acquires;  

15.         if (nextc < 0) // overflow  

16.             throw new Error("Maximum lock count exceeded");  

17.         setState(nextc);  

18.         return true;  

19.     }  

20.     return false;  

21. }  
```

可以看出线程还是在尝试修改state的值（获取锁），注意到tryAcquire的参数是1，也就是将state的值+1，这里为什么不直接将state的值在0到1之间转换呢？

这里就体现ReentrantLock的锁**可重入性**了，在nonfairTryAcquire方法中会判断当前线程是不是锁的owner，如果是就不用再获取锁了，直接将state + 1，所以state的值可能是2、3、4等等，这里还有个细节就是判断state + 1 < 0，也就是超过int的边界了。



回头看addWaiter方法

```
1.  private Node addWaiter(Node mode) {  

2.      Node node = new Node(Thread.currentThread(), mode);  

3.      Node pred = tail;  

4.      if (pred != null) {  

5.          node.prev = pred;  

6.          if (compareAndSetTail(pred, node)) {  

7.              pred.next = node;  

8.              return node;  

9.          }  

10.     }  

11.     enq(node);  

12.     return node;  

13. }  
```

也就是将当前Thread构造成Node放到AQS维护的双向队列的末尾，如果这个双向队列还没有创建（tail == null），先还要初始化。

addWaiter方法返回新插入的Node作为acquireQueued方法的参数



acquireQueued方法

```
1.  final boolean acquireQueued(final Node node, int arg) {  

2.      boolean failed = true;  

3.      try {  

4.          boolean interrupted = false;  

5.          for (;;) {  

6.              final Node p = node.predecessor();  

7.              if (p == head && tryAcquire(arg)) {  

8.                  setHead(node);  

9.                  p.next = null; // help GC  

10.                 failed = false;  

11.                 return interrupted;  

12.             }  

13.             if (shouldParkAfterFailedAcquire(p, node) &&  

14.                 parkAndCheckInterrupt())  

15.                 interrupted = true;  

16.         }  

17.     } finally {  

18.         if (failed)  

19.             cancelAcquire(node);  

20.     }  

21. }  
```

可以看出acquireQueued方法中一直在自旋，当前线程被唤醒后看看自己是不是队列中的第二个节点，如果是就尝试获取锁并将自身设置成头节点，如果不是或者获取失败将自己阻塞。



检查自身是否安全（是否可以阻塞），shouldParkAfterFailedAcquire方法

```
1.  private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {  

2.      int ws = pred.waitStatus;  

3.      if (ws == Node.SIGNAL)  

4.          return true;  

5.      if (ws > 0) {  

6.          do {  

7.              node.prev = pred = pred.prev;  

8.          } while (pred.waitStatus > 0);  

9.          pred.next = node;  

10.     } else {  

11.         compareAndSetWaitStatus(pred, ws, Node.SIGNAL);  

12.     }  

13.     return false;  

14. }  
```

如果当前节点前置节点状态是SIGNAL，就直接阻塞；如果是CANCELLED，就将当前节点连接到前面第一个waitstatus不是CANCELLED的节点（相当于剔除了CANCELLED节点）；其他状态就将前置节点状态设置成SIGNAL



确认应该被阻塞后，调用parkAndCheckInterrupt方法

```
1.  private final boolean parkAndCheckInterrupt() {  

2.      LockSupport.park(this);  

3.      return Thread.interrupted();  

4.  } 
```

LockSupport.park()方法将当前线程挂起到WAITING状态，它需要等待一个中断、unpark方法来唤醒它。



整个lock方法的流程如下

![img](http://pcc.huitogo.club/f614767cc3626ea579d66f5f9d80d652)



上面讲述的就是不公平锁获取到锁的步骤，那么公平锁有什么不一样？**为什么叫公平锁？**

```
1.  final void lock() {  

2.      acquire(1);  

3.  }  
```



可以看出公平锁不允许线程直接尝试设置state的值，而是排队tryAcquire

tryAcquire

```
1.  protected final boolean tryAcquire(int acquires) {  

2.      final Thread current = Thread.currentThread();  

3.      int c = getState();  

4.      if (c == 0) {  

5.          if (!hasQueuedPredecessors() && compareAndSetState(0, acquires)) {  

7.              setExclusiveOwnerThread(current);  

8.              return true;  

9.          }  

10.     }  

11.     else if (current == getExclusiveOwnerThread()) {  

12.         int nextc = c + acquires;  

13.         if (nextc < 0)  

14.             throw new Error("Maximum lock count exceeded");  

15.         setState(nextc);  

16.         return true;  

17.     }  

18.     return false;  

19. }  
```

可以看出关键就在于hasQueuedPredecessors了，就是看看你是不是队列中第二个节点，如果不是，尝试获取锁的机会都没有。



#### 2. unlock方法

```
2.  public void unlock() {  

3.      sync.release(1);  

4.  }  

5.  public final boolean release(int arg) {  

6.      if (tryRelease(arg)) {  

7.          Node h = head;  

8.          if (h != null && h.waitStatus != 0)  

9.              unparkSuccessor(h);  

10.         return true;  

11.     }  

12.     return false;  

13. } 
```

这里release会tryRelease释放当前锁并unparkSuccessor唤醒其他线程去争夺锁



tryRelease方法

```
1.  protected final boolean tryRelease(int releases) {  

2.      int c = getState() - releases;  

3.      if (Thread.currentThread() != getExclusiveOwnerThread())  

4.          throw new IllegalMonitorStateException();  

5.      boolean free = false;  

6.      if (c == 0) {  

7.          free = true;  

8.          setExclusiveOwnerThread(null);  

9.      }  

10.     setState(c);  

11.     return free;  

12. }  
```

这里就是判断state的值是不是0，只有当state为0才将锁的owner置为空，否则就将state - 1，因为之前讲的ReentrantLock锁的可重入性导致state的值可能不为1。



unparkSuccessor方法

```
1.  private void unparkSuccessor(Node node) {  

2.      int ws = node.waitStatus;  

3.      if (ws < 0)  

4.          compareAndSetWaitStatus(node, ws, 0);  

5.      Node s = node.next;  

6.      if (s == null || s.waitStatus > 0) {  

7.          s = null;  

8.          for (Node t = tail; t != null && t != node; t = t.prev)  

9.              if (t.waitStatus <= 0)  

10.                 s = t;  

11.     }  

12.     if (s != null)  

13.         LockSupport.unpark(s.thread);  

14. }
```

这里设置当前节点的waitstatus为0，然后唤醒它的后一个节点，如果后一个节点为空，就往后找第一个waitstatus不为CANCELLED的节点然后唤醒它。这里使用LockSupport.unpark方法将线程的状态变成RUNNING。



#### 3. tryLock方法

tryLock方法相当于直接调用NonfairSync.tryAcquire方法去尝试获取锁，如果获取失败不会将自身加入双向队列并阻塞。

来看一下tryLock(long timeout, TimeUnit unit)怎么实现在规定时间内获取不到锁自动放弃的？

```
1.  public boolean tryLock(long timeout, TimeUnit unit)  

2.          throws InterruptedException {  

3.      return sync.tryAcquireNanos(1, unit.toNanos(timeout));  

4.  }  

5.  public final boolean tryAcquireNanos(int arg, long nanosTimeout)  

6.          throws InterruptedException {  

7.      if (Thread.interrupted())  

8.          throw new InterruptedException();  

9.      return tryAcquire(arg) ||  

10.         doAcquireNanos(arg, nanosTimeout);  

11. }  
```



核心在于doAcquireNanos方法，如果没有马上获取到锁就要执行这个方法

```
1.  private boolean doAcquireNanos(int arg, long nanosTimeout)  

2.          throws InterruptedException {  

3.      if (nanosTimeout <= 0L)  

4.          return false;  

5.      final long deadline = System.nanoTime() + nanosTimeout;  

6.      final Node node = addWaiter(Node.EXCLUSIVE);  

7.      boolean failed = true;  

8.      try {  

9.          for (;;) {  

10.             final Node p = node.predecessor();  

11.             if (p == head && tryAcquire(arg)) {  

12.                 setHead(node);  

13.                 p.next = null; // help GC  

14.                 failed = false;  

15.                 return true;  

16.             }  

17.             nanosTimeout = deadline - System.nanoTime();  

18.             if (nanosTimeout <= 0L)  

19.                 return false;  

20.             if (shouldParkAfterFailedAcquire(p, node) && nanosTimeout > spinForTimeoutThreshold)  

22.                 LockSupport.parkNanos(this, nanosTimeout);  

23.             if (Thread.interrupted())  

24.                 throw new InterruptedException();  

25.         }  

26.     } finally {  

27.         if (failed)  

28.             cancelAcquire(node);  

29.     }  

30. }  
```

这里核心就是使用了一个倒计时的概念，在规定时间内自旋等待获取锁，并且注意到park都是有时间限制的，调用的是LockSupport.parkNanos方法。

关于ReentrantLock的Condition章节要在AQS详情里面讲，因为基本都是在AQS写的。



#### 4. 总结

1. ReentrantLock是基于AQS框架的，主要使用到AQS中的state和双向队列。
2. ReentrantLock有较多的自旋CAS解决了多线程问题。
3. 公平锁的情况优先进入双向队列的线程会被唤醒去获取锁，也有可能会获取失败。
4. ReetrantLock是可重入的独占锁。