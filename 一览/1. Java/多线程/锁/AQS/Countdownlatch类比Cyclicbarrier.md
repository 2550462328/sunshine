先从一个案例分别用CountDownLatch和CyclicBarrier两种方式去实现

场景：很多人开开心心的去旅游，然后出发之前需要集合，等待所有人到达，然后到达之后导游给每个人发旅游说明书，最后出发旅游。



使用CountDownLatch实现

```
1.  public class CountDownLatchTest {  

2.      private ThreadPoolExecutor threadPool =  

3.          new ThreadPoolExecutor(100, 1000, 0, TimeUnit.SECONDS, new LinkedBlockingQueue<Runnable>(100));  

4.      private CountDownLatch downLatch = new CountDownLatch(100);  

5.      public static void main(String[] args) {  

6.          new CountDownLatchTest().doDownLatch();  

7.      }  

8.      private void doDownLatch() {  

9.          for (int i = 0; i < 100; i++) {  

10.             Thread thread = new Thread(() -> {  

11.                 System.out.println(Thread.currentThread().getName() + "：我已到达...");  

12.                *{* downLatch.countDown();  

13.             }, "thread" + i);  

14.             threadPool.execute(thread);  

15.         }  

16.         try {  

17.             downLatch.await();  

18.         } catch (InterruptedException e) {  

19.             e.printStackTrace();  

20.         }  

21.         System.out.println("我是导游，我来分发旅游说明书...");  

22.         System.out.println("所有人已到达，我们出发吧！");  

23.         threadPool.shutdown();  

24.         while(true){  

25.             if(threadPool.isTerminated()){  

26.                 System.out.println("线程池已关闭");  

27.                 break;  

28.             }  

29.         }  

30.     }  

31. }  
```

这里的downLatch.countdown()操作可以在主线程进行也可以在子线程进行，countdown之后还会接着往下走，只会在downLatch.await()地方阻塞



使用CyclicBarrier实现

```
1.  public class CyclicBarrierTest {  

2.      private ThreadPoolExecutor threadPool = new ThreadPoolExecutor(100,1000,0, TimeUnit.SECONDS,new LinkedBlockingQueue<Runnable>(100));  

3.      private  CyclicBarrier barrier = new CyclicBarrier(100,new FinalTask());  

4.      public static void main(String[] args){  

5.          new CyclicBarrierTest().doBarrier();  

6.      }  

7.      private void doBarrier(){  

8.          for(int i = 0 ; i < 100; i++){  

9.              Thread thread = new Thread(()->{  

10.                 System.out.println(Thread.currentThread().getName()+"：我已到达...");  

11.                 try {  

12.                     barrier.await();  

13.                 } catch (InterruptedException e) {  

14.                     e.printStackTrace();  

15.                 } catch (BrokenBarrierException e) {  

16.                     e.printStackTrace();  

17.                 }  

18.                 System.out.println(Thread.currentThread().getName()+"：去出发旅游咯...");  

19.             },"thread"+i);  

20.             threadPool.execute(thread);  

21.         }  

22.         threadPool.shutdown();  

23.         while(true){  

24.             if(threadPool.isTerminated()){  

25.                 System.out.println("线程池已关闭");  

26.                 break;  

27.             }  

28.         }  

29.     }  

30. }  

31. /** 

32.  * 执行解除屏障前的操作，当前线程由最后一个进入屏障的线程执行！ 

33.  * @author ZhangHui 

34.  * @date 2020/3/11 

35.  * @return 

36.  */  

37. class FinalTask  implements Runnable {  

38.     @Override  

39.     public void run() {  

40.         try {  

41.             Thread.sleep(1000);  

42.         } catch (InterruptedException e) {  

43.             e.printStackTrace();  

44.         }  

45.         System.out.println(Thread.currentThread().getName()+"：我是导游，我来分发旅游说明书了...");  

46.     }  

47. }  
```

这里可以看出所有线程都会在barrier.await()处阻塞，直到满足条件才开始继续往下走



通过上面的两个不同实现方式来看一下两者区别

![img](http://pcc.huitogo.club/131c7959685447bcbfadab88b9bef44d)

1. CountDownLatch : **一个线程(或者多个)，等待另外N个线程完成某个事情之后才能执行**。 它像一个计时器，线程完成一个就记一个，就像 报数一样,，只不过是递减的。
2. CyclicBarrier：**N个线程相互等待，任何一个线程没有到达或完成时，所有的线程都必须互相等待**。它像一个水闸，线程执行就像水流，在水闸处都会堵住，等到水满(线程到齐)了，才开始泄流.