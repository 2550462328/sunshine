Atomic相关类的底层实现原理是基于CAS，我们知道CAS有个经典的CAS问题

描述: 第一个线程取到了变量 x 的值 A，然后巴拉巴拉干别的事，总之就是只拿到了变量x 的值 A。这段时间内第二个线程也取到了变量 x 的值 A，然后把变量 x 的值改为B，然后巴拉巴拉干别的事，最后又把变量 x 的值变为 A（相当于还原了）。在这之后第一个线程终于进行了变量 x 的操作，但是此时变量 x的值还是 A，所以 compareAndSet 操作是成功。



所以为了解决这个问题，产生了AtomicMarkableReference和AtomicStampedReference 基于版本号进行标识

- AtomicMarkableReference是我不关心你更新了多少次，只关心了你更没更新
- AtomicStampedReference 是我既关心你更没更新，又关心了更新了多少次

AtomicMarkableReference的版本号是在true和false之间切换的，而AtomicStampedReference 版本号是int类型，可以递增进行标识



可以看出AtomicMarkableReference不能完全规避ABA的问题，只能降低ABA的几率，因为它的原理是更改版本号在true和false之间变动，有可能会出现true--> false --> true的情况，而在另一个线程看来value没有变化，版本号也没有变化，那我是可以进行更改的



使用AtomicStampedReference 规避ABA问题的核心就是线程在使用AtomicStampReference之前要先拿到stamp（版本号），在compareAndSet中进行比较

```
1.  //t1将atomicStampedReference的值从100变成101，然后又变成100.模拟出ABA的问题  

2.  Thread t1 = new Thread(()->{  

3.      try {  

4.          TimeUnit.SECONDS.sleep(1L);  

5.      } catch (InterruptedException e) {  

6.          e.printStackTrace();  

7.      }  

8.      atomicStampedReference.compareAndSet(100,101,atomicStampedReference.getStamp(),atomicStampedReference.getStamp() + 1);  

9.      atomicStampedReference.compareAndSet(101,100,atomicStampedReference.getStamp(),atomicStampedReference.getStamp() + 1);  

10. });  


11. Thread t2 = new Thread(()->{  

12.     int stamp = atomicStampedReference.getStamp();  

13.     try {  

14.         TimeUnit.SECONDS.sleep(2L);  

15.     } catch (InterruptedException e) {  

16.         e.printStackTrace();  

17.     }  

18.     //t2在t1模拟出ABA问题后再去尝试更改atomicStampedReference的值  

19.     // 当然因为版本号不对，不可能修改成功  

20.     boolean isSuccess = atomicStampedReference.compareAndSet(100,101,stamp,stamp + 1);  

21.     System.out.println(isSuccess);  

22. });  


23. t1.start();  

24. t2.start();  
```