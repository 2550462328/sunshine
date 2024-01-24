CyclicBarrierr翻译过来是循环屏障，循环是指这个barrier在使用后（reset或者broken）可以再次使用；屏障是指所有执行await的线程都需要等待到达阀值之后才可以继续执行后面的代码。

**CyclicBarrierr的底层使用的是ReentrantLock和Condition。**



来看一下CyclicBarrierr的源码

#### 1. 内部类Generation

```
2.  private static class Generation {  
3.      boolean broken = false;  
4.  }  
```

Generation代表着一次Barrier的生命周期，里面只有一个属性broken

当broken为true的时候，Barrier被打断，调用await()方法将会抛出BrokenBarrierException异常

当broken为false的时候，Barrier有效

参与CyclicBarrier的线程可能会关联多个Generation，但是只会有一个Generation处于active，其余的不是broken就是tripped了。

**一次Barrier生命周期中Barrier的状态不会影响到下一次生命周期。**



#### 2. 成员变量

```
2.  /** The lock for guarding barrier entry */  

3.  private final ReentrantLock lock = new ReentrantLock();  

4.  /** Condition to wait on until tripped */  

5.  private final Condition trip = lock.newCondition();  

6.  /** The number of parties */  

7.  private final int parties;  

8.  /* The command to run when tripped */  

9.  private final Runnable barrierCommand;  

10. /** The current generation */  

11. private Generation generation = new Generation();  

12. private int count;  
```

- lock：重入锁，是CyclicBarrier线程安全的关键

- trip：条件队列的信号，控制着CyclicBarrier里所有线程的阻塞和唤醒。

- parties：屏障的阀值

- barrierCommand：最后一个进入屏障的线程需要做的事，进一步增加了CyclicBarrier的生命周期

- generation：指向当前CyclicBarrier的生命周期

- count：计数器，记录着到达阀值还需要的数据，范围在parties~0之间




#### 3. 重要方法

##### 3.1 await()

```
1.  public int await() throws InterruptedException, BrokenBarrierException {  

2.      try {  

3.          return dowait(false, 0L);  

4.      } catch (TimeoutException toe) {  

5.          throw new Error(toe); // cannot happen  

6.      }  


7.  }  

8.  private int dowait(boolean timed, long nanos)  

9.      throws InterruptedException, BrokenBarrierException,  

10.            TimeoutException {  

11.     final ReentrantLock lock = this.lock;  

12.     lock.lock();  

13.     try {  

14.         final Generation g = generation;  

15. //如果是broken状态，则抛出BrokenBarrierException异常。该bool值在breakBarrier方法中被设置为true

16.         if (g.broken)  

17.             throw new BrokenBarrierException();  

18. //如果被中断了则设置broken为true，然后抛出InterruptedException异常

19.         if (Thread.interrupted()) {  

20. //将Barrier的当前生命周期的状态标记为不可用

21.             breakBarrier();  

22.             throw new InterruptedException();  

23.         }  

24. //进来一个线程后count会递减，直到0

25.         int index = --count;  

26. //如果index为0，说明有足够多的线程调用了await方法，此时应该放行所有线程

27.         if (index == 0) {  // tripped  

28.             boolean ranAction = false;  

29.             try {  

30.                 final Runnable command = barrierCommand;  

31. //执行放行前的最后一步操作

32.                 if (command != null)  

33.                     command.run();  

34. //如果command.run()出现异常，这里ranAction不会是true的

35.                 ranAction = true;  

36. //进入下一个生命周期

37.                 nextGeneration();  

38.                 return 0;  

39.             } finally {  

40.                 if (!ranAction)  

41.                     breakBarrier();  

42.             }  

43.         }  

44.         //阻塞当前线程直至被中断，打断或者超时

45.         for (;;) {  

46.             try {  

47. //如果没有时间限制，直接将当前线程直接挂起

48.                 if (!timed)  

49.                     trip.await();  

50.                 else if (nanos > 0L)  

51. //如果有时间限制，阻塞一段时间

52.                     nanos = trip.awaitNanos(nanos);  

53.             } catch (InterruptedException ie) {  

54. //当生命周期还是阻塞前的且有效

55.                 if (g == generation && ! g.broken) {  

56.                     breakBarrier();  

57.                     throw ie;  

58.                 } else {  

59.                     Thread.currentThread().interrupt();  

60.                 }  

61.             }  

62.    //如果Barrier被打断，则抛出BrokenBarrierException异常

63.             if (g.broken)  

64.                 throw new BrokenBarrierException();  

65.    //意味着调用await时处于上一个生命周期，而此时却进入了下一个生命周期中

66.             if (g != generation)  

67.                 return index;  

68.    //出现超时的情况

69.             if (timed && nanos <= 0L) {  

70.                 breakBarrier();  

71.                 throw new TimeoutException();  

72.             }  

73.         }  

74.     } finally {  

75.         lock.unlock();  

76.     }  

77. }  
```



CyclicBarrier的主要功能都体现在这儿了，线程是如何阻塞的？以及怎么唤醒？

1. 在阻塞之前先看一下先看下屏障状态是不是broken，当前线程有没有interrupted，屏障有没有到达阀值，注意如果到达阀值的话会执行barrierCommand并且进入下一个生命周期。

2. 如果上面条件都错过，也就是说当前generation是active的，将当前线程阻塞（trip.await()），阻塞根据await方法有没有设置超时条件来分别处理。

3. 在线程被唤醒后也根据正常唤醒和被interrupt分别处理，不正常唤醒会触发breakBarrier操作，使屏障失效。




通过这个方法我们总结一下 barrier.await();的阀口开放除了达到阀值外还可以是

- 有一个线程await超时了
- 有一个await的线程被Interrupt了，前提是确定这个线程是start的
- barrier被reset了，会释放当前所有在await的线程，没有到达await的线程不算
- barrier被broken了,比如上面的reset就会breakBarrier

上述情况**所有线程都抛出BrokenBarrierException**

在finalTask中如果抛出异常所有await的线程也会抛出BrokenBarrierException，程序继续进行



现在回过头来捡漏，看看await方法中的其他方法

##### 3.2 breakBarrier()

**breakBarrier()** 让当前生命周期中的屏障失效

```
1.  private void breakBarrier() {  

2.      generation.broken = true;  

3.      count = parties;  

4.      trip.signalAll();  

5.  }  
```

它会重置count的值，并且唤醒所有处于trip.await()中的线程



##### 3.3 nextGeneration()

**nextGeneration()** 使当前CyclicBarrier进入下一个生命周期

```
1.  private void nextGeneration() {  

2.      // signal completion of last generation  

3.      trip.signalAll();  

4.      // set up next generation  

5.      count = parties;  

6.      generation = new Generation();  

7.  }  
```

这里会new一个Generation，所有会存在多个Generation存在的情况



##### 3.4 reset

**reset()** 类似于强制刷新 = breakBarrier + nextGeneration

```
1.  public void reset() {  

2.      final ReentrantLock lock = this.lock;  

3.      lock.lock();  

4.      try {  

5.          breakBarrier();   // break the current generation  

6.          nextGeneration(); // start a new generation  

7.      } finally {  

8.          lock.unlock();  

9.      }  

10. } 
```



#### 4. 总结

可以看出CyclicBarrier代码还是很简洁的，底层依赖ReentrantLock来实现线程安全，使用Condition来实现阻塞和唤醒，我发现到CyclitBarrier是不会让它里面await的线程回滚的（就是销毁），只要一出现什么意外情况屏障马上失效。

CyclicBarrier适合那种所有线程等待一个条件，只有达到那个条件才继续执行下去的场景。而不像CountDownLatch，每个人都做完一件事了，再等待其他人去做另外一件事。CyclicBarrier强调的是每个人做自己事的时候都等一下，然后继续做手头上的事。