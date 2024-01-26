Executor框架包含三个内容

#### 1. 任务（why）

也就是Executor框架执行的目标，一般是Runnable或Callable



#### 2. 任务执行（how）

任务执行的核心接口Executor接口，继承于它的接口是ExecutorService，具体实现的类是ThreadPoolExecutor和ScheduledThreadPoolExecutor，重要借助任务执行的类是ThreadPoolExecutor。



它们的关系如下：

![img](http://pcc.huitogo.club/b33775e320f54c267ad183f718d4f467)



#### 3. 任务结果（what）

对于异步执行的目标可以用Future接口或者FutureTask类去代表结果。



这里重点看下任务执行，也就是怎么创建线程池去执行任务。

在java.util.concurrent包中，是Jdk并发包得核心，其中有一个重要得类：Executors，他扮演着线程工厂的角色，我们通过Executors创建特定的线程池。

- **newFixedThreadPool**()方法，该方法返回一个固定数量得线程池，该方法得线程数始终保持不变，当有一个任务提交得时候，若线程池空闲，即立即执行，若不空闲，就被暂缓在一个任务队列中等待有空闲得线程去执行。
- **newSingleThreadExecutor**()方法，创建一个线程得线程池，若空闲则执行，若不空闲，就被暂缓在一个任务队列中等待有空闲得线程去执行。
- **newCachedThreadPool**()方法，返回一个可根据实际情况调整线程个数的线程池，不限制最大的线程数量，若有任务，则创建线程，若无任务则不创建线程。如果没有任务，空闲线程默认在60s后回收。
- **newScheduledThreadPool**()方法，该方法返回一个SchededExecutorSerice对象，但该线程可以指定线程的数量。
- **newWorkStealingPool**()方法，创建持有足够线程的线程池来达到快速运算的目的，在内部通 过使用多个队列来减少各个线程调度产生的竞争 。 这里所说的有足够的线程指 JDK 根据 当前线程的运行需求向操作系统申请足够的线程，以保障线程的快速执行，并很大程度 地使用系统资源，提高并发计算的效率，省去用户根据 CPU 资源估算并行度的过程 。 当 然，如果开发者想自己定义线程的并发数，则也可以将其作为参数传入。



做个ScheduledThread的小案例：

```
1.  class Task extends Thread {  

2.      @Override  

3.      public void run() {  

4.          System.out.println(Calendar.getInstance().getTime().getSeconds() + "run");  

5.      }  

6.  }  


7.  public class ScheduledJob {  

8.      public static void main(String[] args) {  

9.          Task task = new Task();  

10.         ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);  

11.         ScheduledFuture<?>  schedulerTask = scheduler.scheduleWithFixedDelay(task, 1, 3, TimeUnit.SECONDS);  

12.     }  

13. }  
```

scheduleWithFixedDelay()里面的参数，第一个任务进程，第二个延时时间，第三个轮询时间，第四个时间单位。



在阿里巴巴的开发手册中强制不允许使用Excutors的方式创建线程池，必须使用下面ThreadPoolExecutor的构造方法

理由如下：

- FixedThreadPool 和 SingleThreadExecutor ： 允许请求的队列长度为 Integer.MAX_VALUE ，**可能堆积大量的请求，从而导致OOM。**
- CachedThreadPool 和 ScheduledThreadPool ： 允许创建的线程数量为 Integer.MAX_VALUE ，**可能会创建大量线程，从而导致OOM。**