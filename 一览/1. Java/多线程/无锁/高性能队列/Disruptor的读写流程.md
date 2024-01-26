先来看下Disruptor是怎么设计的？

1. **底层是环形数组结构，有了ArrayBlockingQueue的优势**
2. **数组长度2^n，方便使用位运算优化内部算法**
3. **无锁设计，采用CAS解决线程安全问题，弥补了ArrayBlockingQueue的劣势**

每个生产者或者消费者线程，会先申请可以操作的元素在数组中的位置，申请到之后，直接在该位置写入或者读取数据。



再看一下Disruptor多线程怎么读写数据

#### 1. 读数据

步骤如下：

1. 申请读取到序号n；
2. 若writer cursor >= n，这时仍然无法确定连续可读的最大下标。从reader cursor开始读取available Buffer，一直查到第一个不可用的元素，然后返回最大连续可读元素的位置；
3. 消费者读取元素。



**读数据如何防止读取的时候，读到还未写的元素？**

Disruptor在多个生产者的情况下，引入了一个与Ring Buffer大小相同的buffer：**available Buffer**。当某个位置写入成功的时候，便把availble Buffer相应的位置置位，标记为写入成功。读取的时候，会遍历available Buffer，来判断元素是否已经就绪。

如下图所示，读线程读到下标为2的元素，三个线程Writer1/Writer2/Writer3正在向RingBuffer相应位置写数据，写线程被分配到的最大元素下标是11。

读线程申请读取到下标从3到11的元素，判断writer cursor>=11。然后开始读取availableBuffer，从3开始，往后读取，发现下标为7的元素没有生产成功，于是WaitFor(11)返回6。

然后，消费者读取下标从3到6共计4个元素。

![img](http://pcc.huitogo.club/d7c706cef2efc9061f24ca2330a3f34d)



#### 2. 写数据

步骤如下：

1. 申请写入m个元素；
2. 若是有m个元素可以写入，则返回最大的序列号。每个生产者会被分配一段独享的空间；
3. 生产者写入元素，写入元素的同时设置available Buffer里面相应的位置，以标记自己哪些位置是已经写入成功的。



**写数据如何防止多个线程重复写同一个元素？也就是写数据的线程安全保障是什么？**

Disruptor的解决方法是，每个线程获取不同的一段数组空间进行操作。这个通过**CAS**很容易达到。只需要在分配元素的时候，通过CAS判断一下这段空间是否已经分配出去即可。



部分代码如下：

```
1.  public long tryNext(int n) throws InsufficientCapacityException  

2.  {  

3.      if (n < 1)  

4.      {  

5.          throw new IllegalArgumentException("n must be > 0");  

6.      }  

7.      long current;  

8.      long next;  

9.      do  

10.     {  

11.         current = cursor.get();  

12.         next = current + n;  

13.         if (!hasAvailableCapacity(gatingSequences, n, current))  

14.         {  

15.             throw InsufficientCapacityException.INSTANCE;  

16.         }  

17.     }  

18.     while (!cursor.compareAndSet(current, next));  

19.     return next;  

20. }  
```



如下图所示，Writer1和Writer2两个线程写入数组，都申请可写的数组空间。Writer1被分配了下标3到下表5的空间，Writer2被分配了下标6到下标9的空间。

Writer1写入下标3位置的元素，同时把available Buffer相应位置置位，标记已经写入成功，往后移一位，开始写下标4位置的元素。Writer2同样的方式。最终都写入完成。

![img](http://pcc.huitogo.club/fe4fe30c962c3906c67cbd801b26b12f)



#### 3. 等待策略

##### 3.1 生产者等待策略

只有一个，LockSupport.parkNanos(1); 休眠1ns



##### 3.2 消费者等待策略

![img](http://pcc.huitogo.club/f1e38f8eba04a99e7f186db434ce44b8)