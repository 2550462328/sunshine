之前学习AtomInteger的时候我们知道对于原子类的底层操作都依赖于sun.misc.Unsafe类来实现，Unsafe调用的方法也就是JNI方法，即本地方法

### 1. CPU中的lock

讲Unsafe的原理之前先了解一下CPU和内存之间的交互关系，在多核心时代下，**多个核心通过同一条总线和内存以及其他硬件进行通信**



下面是CPU和内存的简单的交互过程：

![img](http://pcc.huitogo.club/92e0b3ff32fa5b351bf73a824091df69)

在上图中，CPU 通过两个蓝色箭头标注的总线与内存进行通信。



**如果CPU的多个核心同时对同一片内存进行单次操作，若不加以控制，会导致什么样的错误?**

假设核心1经32位带宽的总线向内存写入64位的数据，核心1要进行两次写入才能完成整个操作。若在核心1第一次写入32位的数据后，核心2从核心1写入的内存位置读取了64位数据。由于核心1还未完全将64位的数据全部写入内存中，核心2就开始从该内存位置读取数据，那么读取出来的数据必定是混乱的。

不过对于这个问题，通过 Intel 开发人员手册，我们可以了解到自奔腾处理器开始，Intel处理器会保证以原子的方式读写按64位边界对齐的四字（quadword）。

根据上面的说明，我们可以知道Intel处理器可以保证单次访问内存对齐的指令以原子的方式执行。



**如果是两次访存的指令呢？**

答案是无法保证。比如递增指令inc dword ptr [...]，等价于DEST = DEST + 1。该指令包含三个操作读->改->写，涉及两次访存。考虑这样一种情况，在内存指定位置处，存放了一个为1的数值。现在 CPU 两个核心同时执行该条指令。两个核心交替执行的流程如下：

```
1.  核心1 从内存指定位置出读取数值1，并加载到寄存器中  

2.  核心2 从内存指定位置出读取数值1，并加载到寄存器中  

3.  核心1 将寄存器中值递减1  

4.  核心2 将寄存器中值递减1  

5.  核心1 将修改后的值写回内存  

6.  核心2 将修改后的值写回内存  
```

经过执行上述流程，内存中的最终值是2，而我们期待的是3，这就出问题了。要处理这个问题，就要避免两个或多个核心同时操作同一片内存区域。

那么怎样避免呢？这就要引入本文的主角 - **lock 前缀**


关于该指令的详细描述，官方文档介绍如下：

```
1.  LOCK—Assert LOCK# Signal Prefix  

2.  Causes the processor’s LOCK# signal to be asserted during   

3.  execution of the accompanying instruction (turns the instruction 

4.  into an atomic instruction). In a multiprocessor environment,   

5.  the LOCK# signal ensures that the processor has exclusive use  

6.  of any shared memory  while the signal is asserted.  
```



从文档中我们可以知道在多处理器环境下，LOCK#信号可以确保**处理器独占使用某些共享内存，**lock 可以被添加在下面的指令前：

ADD, ADC, AND, BTC, BTR, BTS, CMPXCHG, CMPXCH8B, CMPXCHG16B, DEC, INC, NEG, NOT, OR, SBB, SUB, XOR, XADD, and XCHG

通过在 inc 指令前添加 lock 前缀，即让该指令具备原子性。多个核心同时执行同一条inc 指令时，会以串行的方式进行，也就避免了上面所说的那种情况。



**lock 前缀是怎样保证核心独占某片内存区域的呢？**

**1）锁定总线**

通过锁定总线，让某个核心独占使用总线

这种方式的代价就是总线被锁定后，其他核心就不能访问内存了，可能会导致其他核心短时内停止工作。

**2）锁定缓存**

若某处内存数据被缓存在处理器缓存中。处理器发出的 LOCK#信号不会锁定总线，而是锁定缓存行对应的内存区域。其他处理器在这片内存区域锁定期间，无法对这片内存区域进行相关操作



### 2. Unsafe的源码

现在可以看一下Unsafe类的源码

追溯源码如下：

![img](http://pcc.huitogo.club/be3fdd7f240145480046a2b205572012)



Unsafe.java

```
1.  public final class Unsafe {    

2.      public final native boolean compareAndSwapInt(Object o, long offset, int expected, int x);  

3.  } 
```


Unsafe.cpp

```
1.  UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSwapInt(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jint e, jint x))  

2.    UnsafeWrapper("Unsafe_CompareAndSwapInt");  

3.    oop p = JNIHandles::resolve(obj);  // 根据偏移量，计算 value 的地址。这里的 offset 就是 AtomaicInteger 中的 valueOffset  

4.    jint* addr = (jint *) index_oop_from_field_offset_long(p, offset);  // 调用 Atomic 中的函数 cmpxchg，该函数声明于 Atomic.hpp 中  

5.    return (jint)(Atomic::cmpxchg(x, addr, e)) == e;  

6.  UNSAFE_END  
```

Unsafe中对CAS的实现是C++写的，从上图可以看出最后调用的是Atomic:comxchg这个方法，这个方法的实现放在hotspot下的os_cpu包中，说明这个方法的实现和操作系统、CPU都有关系



我们看一下Windows 平台下的 Atomic::cmpxchg 函数

```
1.  inline jint Atomic::cmpxchg (jint exchange_value, volatile jint* dest, jint compare_value) {  InterlockedCompareExchange  

2.    int mp = os::is_MP();  

3.    __asm {  

4.      mov edx, dest  

5.      mov ecx, exchange_value  

6.      mov eax, compare_value    LOCK_IF_MP(mp)  


7.       /* 
8.       * 比较并交换。 
9.       * cmpxchg: 即“比较并交换”指令 
10.      * dword: 全称是 double word，在 x86/x64 体系中，一个 word = 2 byte，dword = 4 byte = 32 bit 
11.      * ptr: 全称是 pointer，与前面的 dword 连起来使用，表明访问的内存单元是一个双字单元 
12.      * [edx]: [...] 表示一个内存单元，edx 是寄存器，dest 指针值存放在 edx 中。 
13.      *    那么 [edx] 表示内存地址为 dest 的内存单元 
14.      *           
15.      * 这一条指令的意思就是，将 eax 寄存器中的值（compare_value）与 [edx] 双字内存单元中的值 进行对比，如果相同，则将 ecx 寄存器中的值（exchange_value）存入 [edx] 内存单元中。 
17.      */  

18.     cmpxchg dword ptr [edx], ecx  

19.   }  


20.   //LOCK_IF_MP源码  

21.   cmp mp, 0  

22.     /* 
23.      * 如果 mp = 0，表明是线程运行在单核 CPU 环境下。此时 je 会跳转到 L0 标记处，  也就是越过 _emit 0xF0 指令，直接执行 cmpxchg 指令。也
25.      * 就是不在下面的 cmpxchg 指令 前加 lock 前缀。 
26.      */  

27.     je L0    /* 

28.      * 0xF0 是 lock 前缀的机器码，这里没有使用 lock，而是直接使用了机器码的形式。 

29.      */   

30.     _emit 0xF0L0:    /*  

31. } 
```

以上代码的核心就是 一条带lock 前缀的 cmpxchg 指令，即l**lock cmpxchg dword ptr [edx], ecx**。



### 3. CAS中ABA的问题

CAS由三个步骤组成，分别是“读取->比较->写回”。考虑这样一种情况，线程1和线程2同时执行CAS 逻辑，两个线程的执行顺序如下：

1. 时刻1：线程1执行读取操作，获取原值 A，然后线程被切换走
2. 时刻2：线程2执行完成 CAS 操作将原值由 A 修改为 B
3. 时刻3：线程2再次执行 CAS 操作，并将原值由 B 修改为 A
4. 时刻4：线程1恢复运行，将比较值（compareValue）与原值（oldValue）进行比较，发现两个值相等。 然后用新值（newValue）写入内存中，完成 CAS 操作

虽然线程1的CAS操作成功，但是整个过程就是有问题的。比如链表的头在变化了两次后恢复了原值，但是不代表链表就没有变化。



JAVA中提供了AtomicStampedReference/AtomicMarkableReference来处理会发生ABA问题的场景，主要是在对象中额外再增加一个标记来标识对象是否有过变更。