先来看一下ThreadPoolExecutor中的重要成员，除了上面绑定线程池状态和线程数的ctl。

```
    //线程池任务队列，用于存放等待执行的任务  
1.  private final BlockingQueue<Runnable>  workQueue;   

    //用于线程池一些统计信息更新时的锁，e.g.如下的largestPoolSize，complectedTaskNum,ctl的状态和线程数更新时。
2.   private final ReentrantLock mainLock = new ReentrantLock();
    
    //线程池工作集，Runnable的实现，里面封装了thread变量成员，用于执行任务  
3.   private final HashSet<Worker> workers = new HashSet<Worker>();
    
    //客户端调用awaitTerminate()时，将阻塞与此，线程池处于terminate后，会调用condition.signalAll()通知（有time参数的另外考虑）。
4.   private final Condition termination = mainLock.newCondition();
   
    //用于记录线程池运行过程曾出现过得最大线程数，不同于maximumPoolSize  
6.   private int largestPoolSize;  
	
    //用于动态记录线程池完成的任务数  
7.   private long completedTaskCount;  
		
    //用于创建新的线程,newThread(Runnable).  
8.   private volatile ThreadFactory threadFactory; 
		
     //拒绝任务提交的处理器  
9.   private volatile RejectedExecutionHandler handler; 
		
    //当线程数大于corePoolsize时，多出那部分线程处于idle状态时，最大保活时间。  
10.  private volatile long keepAliveTime; 
		
    //当线程数小于corePoolsize时，是否允许其也运用保活时间限制。  
11.  private volatile boolean allowCoreThreadTimeOut;  

    //核心池的大小（即线程池中的线程数目大于这个参数时，提交的任务会被放进任务缓存队列）
12.  private volatile int corePoolSize;   

     //线程池运行的最大线程数。  
13.  private volatile int maximumPoolSize;
```


因为threadPool.submit(thread)的底层也是threadPool.execute(thread)

所以重点看**execute**()的源码

```
1.  public void execute(Runnable command) {  

2.          if (command == null)  

3.              throw new NullPointerException();  

4.          int c = ctl.get();//读取ctl变量  

5.          if (workerCountOf(c) < corePoolSize) {   //当前线程数和corePoolSize比较，当小于时：  

6.              if (addWorker(command, true)) //addWorker一个new thread来处理该任务(true的情况)，直接返回；如果addWork返回false(线程池被shutdown or shutdownNow;或者同时又别的客户端提交任务，并且此时线程数大于corePoolSize);继续往下执行  

9.                  return;  

10.             c = ctl.get(); //重新读取ctl变量  

11.         }  

12.         if (isRunning(c) && workQueue.offer(command)) {//对addWorker返回false的情况进行判断,当线程池还运行着，说明是因为thread number 大于corePoolSize的情况，则&&操作第二个表达式把任务添加到workQueue队列中  

15.             int recheck = ctl.get(); //再次读取ctl，防止并发

16.             if (! isRunning(recheck) && remove(command))   //把任务加入队列的同时，pool被shutdown， 则从队列中删除task,并且调用rejectHandler的方法  

17.                 reject(command);  

18.             else if (workerCountOf(recheck) == 0)  //如果此时线程数为0（即设置了allowCorePoolSizeTimeOut为true的情况），则追加一个new thread(初始任务为null)  

19.                 addWorker(null, false); 

20.         }  

21.         else if (!addWorker(command, false)) //对于其他addWorker()为false的情况，即！isRunning和 workQueue.offer()失败的情况，再次尝试addWorker()添加new thread，如果为false,说明pool被关闭或者达到pool饱和，直接reject

24.             reject(command);  

25.     }
```



execute的执行流程如下图：

![img](http://pcc.huitogo.club/035041727e4daa9f76397ec32f4191cf)



execute的过程可以分三步

第一步：当前工作线程数是否小于corePoolSize，即是否还能new线程去执行这个Runnable。

第二步：没有足够线程处理了就放入workQueue中，等待corePoolSize下的线程执行完毕，在此过程中要考虑有其他任务加进来影响了当前工作线程数的值和判断线程池是否被强行关闭了，这时候需要从workQueue中移除且执行拒绝策略；如果设置了allowCorePoolSizeTimeOut为true，即不用等待从workQueue中去取Runnable了，直接new一个线程去处理当前这条Runnable。

第三步：执行第三步是因为第二步中的workQueue满了或者执行第二步时线程池被关闭了这时候仍然努力去new一个线程执行这个Runnable，如果返回false则执行reject方法。



#### 1. addWorker

这里注意一下上面三次addWorker的不同含义

1. addWorker(command, true)：正常范围内添加worker并指定firstTask
2. addWorker(null, false)：由于允许线程池消除空闲worker的前提下，worker工作组没有线程了，那就起一个空的worker去执行吧
3. addWorker(command, false)：向workQueue中添加task失败了，说明线程池太忙了，那么允许线程池在maximuxPoolSize的范围内新建Worker去分忧。如果这都失败了，那就只能执行reject策略了。



execute()中的**reject**方法

```
1.  final void reject(Runnable command) {  

2.      handler.rejectedExecution(command, this); //调用rejectHander的处理方法处理。  

3.  } 
```



execute()中的**addWorker**方法

```
1.  private boolean addWorker(Runnable firstTask, boolean core) {  

2.      retry:  

3.      for (;;) {  

4.          int c = ctl.get();  

5.          int rs = runStateOf(c);  

6.          if (rs >= SHUTDOWN &&   ! (rs == SHUTDOWN &&   firstTask == null &&   ! workQueue.isEmpty()))   // 判断还有没有必要在新建worker去执行task  

10.             return false;  

11.         for (;;) {  

12.             int wc = workerCountOf(c);  

13.             if (wc >= CAPACITY ||  wc >= (core ? corePoolSize : maximumPoolSize)) // pool达到饱满，放弃新建worker  

15.                 return false;  

16.             if (compareAndIncrementWorkerCount(c)) // cas增加worker数量  

17.                 break retry;  // 流程结束  

18.             c = ctl.get();  // cas增加失败后  

19.             if (runStateOf(c) != rs)  // 如果线程池状态发生变化，回到开始判断是否有必要新增worker的地方  

20.                 continue retry;  

21.         }  

22.     }  

23.     boolean workerStarted = false;  // worker是否已经启动  

24.     boolean workerAdded = false; // worker是否已经加入了workers  

25.     Worker w = null;  

26.     try {  

27.         w = new Worker(firstTask);  

28.         final Thread t = w.thread;  

29.         if (t != null) {  

30.             final ReentrantLock mainLock = this.mainLock;   

31.             mainLock.lock(); // 加锁处理  

32.             try {  

33.                 int rs = runStateOf(ctl.get());  

34.                 if (rs < SHUTDOWN ||  (rs == SHUTDOWN && firstTask == null)) { // 这里一种是线程池还在运行，一种是线程池即将关闭了而且当前worker也没有task，那么就没有必要新增worker线程了  

36.                     if (t.isAlive()) // 判断worker的线程状态  

37.                         throw new IllegalThreadStateException();  

38.                     workers.add(w); // 新增worker  

39.                     int s = workers.size();  

40.                     if (s > largestPoolSize) // 记录线程池运行过的最大的线程数  

41.                         largestPoolSize = s;  

42.                     workerAdded = true;  

43.                 }  

44.             } finally {  

45.                 mainLock.unlock(); // 解锁  

46.             }  

47.             if (workerAdded) { // 启动worker  

48.                 t.start(); // 这里有可能出现OOM，导致workerStarted=false  

49.                 workerStarted = true;  

50.             }  

51.         }  

52.     } finally {  

53.         if (! workerStarted) // 如果出现OOM，进行失败处理  

54.             addWorkerFailed(w);  

55.     }  

56.     return workerStarted;  

57. }  
```



上述流程步骤

1. 判断有没有必要去新建一个worker（线程）去执行firstTask和workQueue中的task
2. 确实有必要就cas增加worker的数量
3. 再实例化一个worker，将它加入workers工作组，启动这个worker
4. 如果启动失败了，进行失败处理，失败处理代码看下面



addWorker中的**addWorkerFailed**方法

```
1.  private void addWorkerFailed(Worker w) {  

2.      final ReentrantLock mainLock = this.mainLock;  

3.      mainLock.lock();  

4.      try {  

5.          if (w != null)  

6.              workers.remove(w); // 从workers工作组移除失败的worker  

7.          decrementWorkerCount(); // 减少workers的数量  

8.          tryTerminate(); //检查线程池有没有terminate

9.      } finally {  

10.         mainLock.unlock();  

11.     }  

12. } 
```



#### 2. tryTerminate

失败处理的逻辑也分三步，这里看一下**tryTerminate**的方法

```
1.  final void tryTerminate() {  

2.      for (;;) {  

3.          int c = ctl.get();  

4.          // 如果线程池在运行或者已经terminate了或者正在shutdown且还有task没完成 这些情况下不能强行terminate  

5.          if (isRunning(c) ||  runStateAtLeast(c, TIDYING) ||   (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))  

8.              return;  

9.          if (workerCountOf(c) != 0) { // 将workers中的worker线程给interrupt掉  

10.             interruptIdleWorkers(ONLY_ONE); // ONLY_ONE代表是否只interrupt掉一个worker  

11.             return;  

12.         }  

13.         // 这里说明当前线程池没有线程运行，开始terminate掉线程池  

14.         final ReentrantLock mainLock = this.mainLock;  

15.         mainLock.lock();  

16.         try {  

17.             if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) { // 线程池状态改为tidying，此时离terminate状态只差临门一脚  

18.                 try {  

19.                     terminated(); // 这就是临门一脚  

20.                 } finally {  

21.                     ctl.set(ctlOf(TERMINATED, 0)); // 线程池状态改为terminated  

22.                     termination.signalAll(); // 通知那些阻塞awaitTerminate（）方法客户端。  

23.                 }  

24.                 return;  

25.             }  

26.         } finally {  

27.             mainLock.unlock();  

28.         }  

29.         // 到这儿了说明没有cas更改线程池状态成功，那么下一次吧  

30.     }  

31. }  
```

上述过程分两步，先interrupt掉workers中的worker线程，再terminate线程池。



#### 3. runWorker

看完了addWorker中worker添加失败后的逻辑后，回头看一下在worker添加成功后发生了，也就是worker.run方法，实际调用的是**runWorker**方法，代码如下

```
1.  final void runWorker(Worker w) {  

2.      Thread wt = Thread.currentThread();  

3.      Runnable task = w.firstTask;  

4.      w.firstTask = null;   

5.      w.unlock(); // allow interrupts  

6.      boolean completedAbruptly = true; // 这个代表worker执行任务中有没有意外暴毙  

7.      try {  

8.          while (task != null || (task = getTask()) != null) { // 如果worker自身接的任务是Null，就从workQueue中拿任务  

9.              w.lock();  
                   
		// 做任务之前，看看线程池是不是正在terminate，如果是的话就自我毁灭（interrupt）
10.             if ((runStateAtLeast(ctl.get(), STOP) ||   (Thread.interrupted() &&   runStateAtLeast(ctl.get(), STOP))) &&   !wt.isInterrupted())  

14.                 wt.interrupt();  

15.             try {  

16.                 beforeExecute(wt, task); // 这个方法用来覆盖  

17.                 Throwable thrown = null;  

18.                 try {  

19.                     task.run();  

20.                 } catch (RuntimeException x) {  

21.                     thrown = x; throw x;  

22.                 } catch (Error x) {  

23.                     thrown = x; throw x;  

24.                 } catch (Throwable x) {  

25.                     thrown = x; throw new Error(x);  

26.                 } finally {  

27.                     afterExecute(task, thrown); // 这个方法用来覆盖  

28.                 }  

29.             } finally {  

30.                 task = null;  

31.                 w.completedTasks++; //worker完成任务数 + 1  

32.                 w.unlock();  

33.             }  

34.         }  

35.         completedAbruptly = false;  

36.     } finally {  

37.         processWorkerExit(w, completedAbruptly); // 收尾工作，根据当前的worker是否正常退出  

38.     }  

39. }  
```

上述过程就是

worker线程执行自身任务以及从workerQueue中取出task去执行，直到没有任务可执行或者执行任务中出现异常时退出，退出会执行一次收尾工作。



#### 4. getTask

先看一下怎么从workerQueue中取出task，方法**getTask**代码如下

```
1.  private Runnable getTask() {  

2.      boolean timedOut = false; // timedOut代表上一次循环取task的时候是否超时了  

3.      for (;;) {  

4.          int c = ctl.get();  

5.          int rs = runStateOf(c);  

6.          // 如果线程已经在terminate或者在stop中但是workQueue中没有任务的时候直接减少worker的数量  

7.          if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {  

8.              decrementWorkerCount();  

9.              return null;  

10.         }  

11.         int wc = workerCountOf(c);  

12.         // allowCoreThreadTimeOut 表示是否允许在worker数量没有达到corePoolSize的时候就消除空间worker  

13.         // timed 的含义就是当前是否可以消除空闲worker  

14.         boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;  

15.         // 这里判断当前的worker是否有点多余  

16.         // 如果当前工作的woker数量大于maximumPoolSize了，那肯定是多余了  

17.         // 在当前worker没有任务，就是上一次没取到任务并且当前线程池允许消除空闲worker的前提下，如果workers工作组里面至少有一个线程了，或者没有任务了，那当前worker就是多余的  

18.         if ((wc > maximumPoolSize || (timed && timedOut))  && (wc > 1 || workQueue.isEmpty())) {  

20.             if (compareAndDecrementWorkerCount(c))  

21.                 return null;  

22.             continue;  

23.         }  

24.         // 开始接取任务，分两种  

25.         // 一种是允许线程池消除空闲worker，那么如果这个worker在keepAliveTime时间内没有拿到task，说明现在线程池不忙，下一次循环考虑清除你  

26.         // 另一种不允许的情况下也就是线程数没有限制（小于maximumPoolSize）的情况下，那无所谓了，你就一直等着吧，等到有task进workQueue了你就拿出来执行  

27.         try {  

28.             Runnable r = timed ?   workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :  workQueue.take();  

31.             if (r != null)  

32.                 return r;  

33.             timedOut = true;  

34.         } catch (InterruptedException retry) {  

35.             timedOut = false;  

36.         }  

37.     }  

38. }  
```



上述过程步骤如下

1. 如果线程已经在stop或者在shutdown中但是workQueue中没有任务的时候直接减少worker的数量 

2. 在线程池没有stop或者shutdown但是还有task需要执行的时候判断当前这个worker是不是多余的

3. 如果当前工作的woker数量大于maximumPoolSize了，那肯定是多余了 

4. 在当前worker没有任务，就是上一次没取到任务并且当前线程池允许消除空闲 worker的前提下，如果workers工作组里面至少有一个线程了，或者没有任务了，那当 前worker就是多余的 

5. 开始接取任务，分两种

   1）一种是允许线程池消除空闲worker，那么如果这个worker在keepAliveTime时间内没有拿到task，说明现在线程池不忙，下一次循环考虑辞退你 

   2）另一种不允许的情况下也就是线程数没有限制（小于maximumPoolSize）的情况下，那无所谓了，你就一直等着吧，等到有task进workQueue了你就拿出来执行



#### 5. processWorkerExit

讲完woker线程怎么从workQueue中取task之后，再回到runWorker中看一下woker执行完成或者失败后的收尾工作

runWorker()中方法**processWorkerExit**代码如下

```
1.  private void processWorkerExit(Worker w, boolean completedAbruptly) {  

2.      // 如果worker是暴毙的话，是不会调整workers的数量的，这里手动调整  

3.      if (completedAbruptly)   

4.          decrementWorkerCount();  

5.      final ReentrantLock mainLock = this.mainLock;  

6.      mainLock.lock();  

7.      try {  

8.          // 统计worker的工作量  

9.          completedTaskCount += w.completedTasks;  

10.         // 辞退worker  

11.         workers.remove(w);  

12.     } finally {  

13.         mainLock.unlock();  

14.     }  

15.     // 看看线程池是不是在terminated中  

16.     tryTerminate();  

17.     // 辞退worker后开始考虑一个问题了，我这个线程池还有没有活干  

18.     // 有活干得话，如果刚刚worker是暴毙得话，那我肯定要再雇一个wokrer来  

19.     // 如果刚刚worker不是暴毙的，是下岗的话，那我要确认一下现在忙不忙，忙的话还得再请worker，不忙的话一个也够用了  

20.     int c = ctl.get();  

21.     if (runStateLessThan(c, STOP)) {  

22.         if (!completedAbruptly) {  

23.             int min = allowCoreThreadTimeOut ? 0 : corePoolSize;  

24.             if (min == 0 && ! workQueue.isEmpty())  

25.                 min = 1;  

26.             if (workerCountOf(c) >= min)  

27.                 return;  // replacement not needed  

28.         }  

29.         addWorker(null, false);  

30.     }  

31. } 
```



上述过程如下

1. 如果worker是暴毙的话那我要手动调整workers的数量
2. 统计一下worker的工作（发放一下工资），再辞退它
3. 辞退worker后开始考虑一个问题了，我这个线程池还有没有活干 
4. 有活干的话，如果刚刚worker是暴毙得话，那我肯定要再雇一个wokrer来 
5. 如果刚刚worker不是暴毙的，是下岗的话，那我要确认一下现在忙不忙，忙的话还得再请worker，不忙的话一个也够用了