#### 1. synchronized同步代码块

```
3.  public class SynchronizedDemo {  

4.      public void method() {  

5.          synchronized (this) {  

6.              System.out.println("synchronized 代码块");  

7.          }  

8.      }  

9.  }  
```



使用javap -c -s -v -l 查看一下SynchronizedDemo .class的字节码信息

![img](http://pcc.huitogo.club/729beb14cdf0ec36880b8968436467df)



从上面我们可以看出：

在进入同步代码块之前有个monitorenter指令，它指向同步代码块的开始位置；离开同步代码块有个monitorexit指令，它指向同步代码块的结束位置。monitorenter和monitorexit指令都需要一个Object类型的参数来指明要锁定和解锁的对象。

调用monitorenter指令的时候，线程会去尝试获取monitor（monitor存在于每个对象的对象头中，对象头中有一个名为MarkWord的部分，该部分记录了对象和锁有关的信息）的所有权，当计数器为0的时候代表可以获取，获取后将锁的计数器设为1。相应的调用monitorexit指令的时候将锁的计数器设为0。如果获取对象锁失败，当前线程进入阻塞状态，直到锁被另一个线程释放。

monitor依赖操作系统的 MutexLock( 互斥锁 来实现的 线程被阻塞后便进入内核（ Linux ）调度状态，这个会导致系统在用户态与内核态之间来回切换，严重影响锁的性能。



![img](http://pcc.huitogo.club/d2e9b8d83a0343ca7d4403ac4361a843)

#### 2. synchronized同步方法

```
2.  public class SynchronizedDemo2 {  

3.      public synchronized void method() {  

4.          System.out.println("synchronized 方法");  

5.      }  

6.  }  
```



查看对应的字节码信息

![img](http://pcc.huitogo.club/66ad861042e8ef5a5e89b909920d66a3)



可以看到并没有monitorenter和monitorexit指令，取而代之的是ACC_SYNCHRONIZED标识，该标识指明了该方法是一个同步方法，JVM通过该 ACC_SYNCHRONIZED访问标志来辨别一个方法是否声明为同步方法，从而执行相应的同步调用。