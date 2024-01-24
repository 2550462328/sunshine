#### 1. await

CountDownLatch是一个计时器，在计时器没有到达0的时候，所有调用await的线程都会被阻塞，被阻塞的线程去哪儿了，当然是变成了Node放到AQS的同步队列了。看源码吧

```
1.  public void await() throws InterruptedException {  

2.      sync.acquireSharedInterruptibly(1);  

3.  }  

4.  public final void acquireSharedInterruptibly(int arg)  

5.          throws InterruptedException {  

6.      if (Thread.interrupted())  

7.          throw new InterruptedException();  

8.      if (tryAcquireShared(arg) < 0)  

9.          doAcquireSharedInterruptibly(arg);  

10. }  
```



CountDownLatch中重写的tryAcquireShared非常简单，就是判断state ==0

```
1.  protected int tryAcquireShared(int acquires) {  

2.      return (getState() == 0) ? 1 : -1;  

3.  }  
```



state的值怎么来的呢，看CountDownLatch的构造方法

```
1.  public CountDownLatch(int count) {  

2.      if (count < 0) throw new IllegalArgumentException("count < 0");  

3.      this.sync = new Sync(count);  

4.  }  

5.  Sync(int count) {  

6.        setState(count);  

7.  }  
```

就是将我们定义的countDownLatch的阀值作为state的值，在AQS里面我们可以将这个state称之为“资源”更好理解。AQS是一个框架，不会想着你去实现什么，它只关心你能不能获取到资源，获取到资源之后怎么做。



回头重点看下doAcquireSharedInterruptibly方法，顾名思义它是会响应中断的

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

注意这里引入了一个新的Node类型SHARED。



整体逻辑和我们之前学ReentrantLock的时候探索到的AQS独占式的源码差不多。那么一个线程它获取不到资源的时候会怎么做呢？

加入AQS的同步队列 --> 将自己放到安全的位置并阻塞 -->在被唤醒的时候查看自己是不是队列中第二个节点-->如果是老二就去竞争资源-->拿到资源后成为头节点并唤醒后面的节点

这里可以类比一下和之前AQS独占式acquire的代码了，独占式在拿到资源后只会setHead()，就是把自己激活去运行，但是咱们共享式大兄弟就不会了，它还会提醒后面的兄弟们setHeadAndPropagate()，果然是好兄弟啊。



好兄弟归好兄弟，人家是有前提的，看看前提吧

```
1.  private void setHeadAndPropagate(Node node, int propagate) {  

2.      Node h = head; // Record old head for check below  

3.      setHead(node);  

4.      if (propagate > 0 || h == null || h.waitStatus < 0 ||  

5.          (h = head) == null || h.waitStatus < 0) {  

6.          Node s = node.next;  

7.          if (s == null || s.isShared())  

8.              doReleaseShared();  

9.      }  

10. }  
```

这里propagate 代表资源，前提就是**还有剩余资源**，或者新老head节点为空，或者新老节点处于signal状态。



看看是怎么通知后面的兄弟们吧，doReleaseShared方法

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

最终的目的是unpark所有的后继节点，前面的节点都变成propagate 。

当h==head的时候说明链表循环结束了，没有后继节点可以通知了。



好了，来理清一下await的流程

如果state!=0的话，就将当前线程加入AQS同步队列阻塞起来，在被唤醒的时候如果拿到了资源会唤醒整个链表。



#### 2. countDown

countDown的意思很简单了，就是去修改state的值，在state降到0的时候就去唤醒整个链表

```
1.  public void countDown() {  

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

这里唤醒整个链表还是用到了await里面的doReleaseShared方法。



#### 3. 总结

1. countdownLatch底层是AQS的共享模式。
2. AQS的共享式跟独占式的区别就在于一个线程拿到资源后会不会去通知后继节点（整个链表）。
3. 在AQS里面是从资源的角度去实现的，AQS是一个框架，不会想着你去实现什么，它只关心你能不能获取到资源，获取到资源之后怎么做，以及释放资源后会发生什么。