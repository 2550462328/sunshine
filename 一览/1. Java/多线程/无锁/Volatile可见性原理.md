关于volatile的一个漫长的故事，先总结一下：**通过内存屏障禁止指令重排序保证多线程环境下的共享数据的可见性、安全性**



一台计算机中最核心的组件是CPU 、内存、以及 I/O设备。在整个计算机的发展历程中，除了CPU、内存以及IO设备不断迭代升级来提升计算机处理性能之外，还有一个非常核心的矛盾点，就是这三者在处理速度的差异。CPU的计算速度是非常快的，内存次之最后是IO设备比如磁盘。而在绝大部分的程序中，一定会存在内存访问，有些可能还会存在I/O设备的访问为了提升计算性能，CPU从单核升级到了多核甚至用到了超线程技术最大化提高CPU的处理性能，但是仅仅提升CPU性能还不够，如果后面两者的处理性能没有跟上，意味着整体的计算效率取决于最慢的设备。为了平衡三者的速度差异，最大化的利用CPU提升性能，从硬件、操作系统、编译器等方面都做出了很多的优化

1. CPU增加了高速缓存
2. 操作系统增加了进程、线程。通过CPU的时间片切换最大化的提升CPU的使用率
3. 编译器的指令优化，更合理的去利用好CPU的高速缓存然后每一种优化，都会带来相应的问题，而这些问题也是导致线程安全性问题的根源。为了了解前面提到的可见性问题的本质，我们有必要去了解



**CPU高速缓存**

线程是CPU调度的最小单元，线程设计的目的最终仍然是更充分的利用计算机处理的效能，但是绝大部分的运算任务不能只依靠处理器计算就能完成，处理器还需要与内存交互，比如读取运算数据、存储运算结果，这个I/O操作是很难消除的。而由于计算机的存储设备与处理器的运算速度差距非常大，所以现代计算机系统都会增加一层读写速度尽可能接近处理器运算速度的高速缓存来作为内存和处理器之间的缓冲：将运算需要使用的数据复制到缓存中，让运算能快速进行，当运算结束后再从缓存同步到内存之中。

通过高速缓存的存储交互很好的解决了处理器与内存的速度矛盾，但是也为计算机系统带来了更高的复杂度，因为它引入了一个新的问题，**缓存一致性**。



**什么叫缓存一致性呢？**

首先，有了高速缓存的存在以后，每个CPU的处理过程是，先将计算需要用到的数据缓存在CPU高速缓存中，在CPU进行计算时，直接从高速缓存中读取数据并且在计算完成之后写入到缓存中。在整个运算过程完成后，再把缓存中的数据同步到主内存。由于在多CPU中，每个线程可能会运行在不同的CPU内，并且每个线程拥有自己的高速缓存。同一份数据可能会被缓存到多个CPU中，如果在不同CPU中运行的不同线程看到同一份内存的缓存值不一样就会存在缓存不一致的问题为了解决缓存不一致的问题，在CPU层面做了很多事情，主要提供了两种解决办法：

1. **总线锁**
2.  **缓存锁**
       

总线锁，简单来说就是，在多cpu下，当其中一个处理器要对共享内存进行操作的时候，在总线上发出一个 LOCK信号，这个信号使得其他处理器无法通过总线来访问到共享内存中的数据， 总线锁把CPU和内存之间的通信锁住了，这使得锁定期间，其他处理器不能操作其内存地址的数据，所以总线锁定的开销比较大 。

这种机制显然是不合适的如何优化呢？最好的方法就是控制锁的保护粒度，我们只需要保证对于被多个CPU缓存的同一份数据是一致的就行。所以引入了缓存锁，它的核心机制是基于缓存一致性协议来实现的。



**MESI 协议**

为了达到数据访问的一致，需要各个处理器在访问缓存时遵循一些协议，在读写时根据协议来操作，常见的协议有MSI MESI MOSI等。 最常见的就是 MESI 协议。



MESI表示缓存行的四种状态，分别是
1. M(Modify)表示共享数据只缓存在当前CPU缓存中,并且是被修改状态，也就是缓存的数据和主内存中的数据不一致

2. E(Exclusive) 表示缓存的独占状态，数据只缓存在当前CPU 缓存中，并且没有被修改

3. S(Shared) 表示数据可能被多个 CPU 缓存，并且各个缓存中的数据和主内存数据一致

4. I(Invalid) 表示缓存已经失效

   

在MESI 协议中，每个缓存的控制器不仅知道自己的读写操作，而且也监听其它Cache的读写操作

对于MESI协议，从CPU读写角度来说会遵循以下原则:

1. CPU读请求：缓存处于M、E、S状态都可以被读取，I状态CPU只能从主存中读取数据
2. CPU写请求：缓存处于M、E状态才可以被写。对于S状态的写，需要将其他CPU中缓存行置为无效才可写



**总结可见性的本质**

由于CPU高速缓存的出现使得 如果多个cpu同时缓存了相同的共享数据时，可能存在可见性问题。也就是CPU0修改了自己本地缓存的值对于CPU1不可见。不可见导致的后果是CPU1后续在对该数据进行写入操作时，是使用的脏数据。使得数据最终的结果不可预测。实际上，这种情况很难模拟。因为我们无法让某个线程指定某个特定CPU，这是系统底层的算法JVM应该也是没法控制的。还有最重要的一点，就是你无法预测CPU缓存什么时候会把值传给主存，可能这个时间间隔非常短，短到你无法观察到。最后就是线程的执行的顺序问题，因为多线程你无法控制哪个线程的某句代码会在另一个线程的某句代码后面马上执行。所以我们只能基于它的原理去了解这样一个存在的客观事实。



了解到这里，大家应该会有一个疑问，刚刚不是说基于缓存一致性协议或者总线锁能够达到缓存一致性的要求吗？

**为什么还需要加volatile关键字？或者说为什么还会存在可见性问题呢？**



**MESI优化带来的可见性问题**

MESI协议虽然可以实现缓存的一致性，但是也会存在一些问题。就是各个CPU缓存行的状态是通过消息传递来进行的。 如果CPU0要对一个在缓存中共享的变量进行写入，首先需要发送一个失效的消息给到其他缓存了该数据的CPU 。并且要等到他们的确认回执。CPU0在这段时间内都会处于**阻塞状态**。为了避免阻塞带来的资源浪费。在cpu中引入了Store Bufferes。CPU0只需要在写入共享数据时，直接把数据写入到storebufferes中同时发送invalidate消息，然后继续去处理其他指令。当收到其他所有CPU发送了 nvalidate acknowledge消息时再将store bufferes中的数据存储至cache line中。最后再从缓存行同步到主内存。



  但是这种优化存在两个问题
       1. 数据什么时候提交是不确定的 ，因为需要等待其他 cpu给回复才会进行数据同步。这里其实是一个异步操作
       2. 引入了storebufferes后，处理器会先尝试从storebuffer中读取值，如果store buffer中有数据，则直接从storebuffer中读取，否则就再从缓存行中读取



看一个例子

```
public void cpu0() {
        value = 99;
        change = true;
    }
public void cpu1() {
        if (change) {
            value = 99;
        }
    }
```

cpu0和cpu1分别在两个独立的CPU上执行。假如cpu0的缓存行中缓存了change这个共享变量,并且状态为E而Value可能是S状态。那么这个时候，cpu0在执行的时候，会先把value=99 的指令写入到 store buffer 中。并且 通知给其他缓存了该value变量的CPU.在等待其他 CPU通知结果的时候，cpu0会继续执行change=true 这个指令.而因为当前cpu0缓存了change并且是 Exclusive状态所以可以直接修改change=true.这个时候cpu1发起read操作去读取change的值可能为true ，但是 value 的值不等于99，这种情况我们可以认为是cpu的乱序执行，也可以认为是一种重排序，而这种重排序会带来可见性的问题.所以在cpu层面提供了memorybarrier(内存屏障的指令，从硬件层面来看这个 memroybarrier就是CPU flush store bufferes中的指令。软件层面可以决定在适当的地方来插入内存屏障。



**CPU的内存屏障问题**

什么是内存屏障从前面的内容基本能有一个初步的猜想，内存屏障就是将store bufferes中的指令写入到内存，从而使得其他访问同一共享内存的线程的可见性。X86的 memory barrier读屏障、写屏障、全屏障。



内存屏障，也叫内存栅栏，是一种CPU指令，用于控制特定条件下的重排序和内存可见性问题。

1. LoadLoad屏障：对于这样的语句Load1; LoadLoad; Load2，在Load2及后续读取操作要读取的 数据被访问前，保证Load1要读取的数据被读取完毕。
2. StoreStore屏障：对于这样的语句Store1; StoreStore; Store2，在Store2及后续写入操作执行 前，保证Store1的写入操作对其它处理器可见。
3. LoadStore屏障：对于这样的语句Load1; LoadStore; Store2，在Store2及后续写入操作被刷出 前，保证Load1要读取的数据被读取完毕。
4. StoreLoad屏障：对于这样的语句Store1; StoreLoad; Load2，在Load2及后续所有读取操作执行 前，保证Store1的写入对所有处理器可见。它的开销是四种屏障中最大的。 在大多数处理器的实 现中，这个屏障是个万能屏障，兼具其它三种内存屏障的功能。



StoreMemoryBarrier(写屏障)告诉处理器在写屏障之前的所有已经存储在存储缓存(store中的数据同步到主内存，简单来说就是使得写屏障之前的指令的结果对屏障之后的读或者写是可见的LoadMemoryBarrier(读屏障)

处理器在读屏之后的读操作 都在读屏障之后执行。配合写屏障，使得写屏障之前的内存更新对于读屏障之后的读操作是可见的FullMemoryBarrier(全屏障)确保屏障前的内存读写操作的结果提交到内存之后，再执行屏障后的读写操作有了内存屏障以后。

对于上面这个例子，我们可以这么来改，从而避免出现可见性问题：

```
public void cpu0() {
        value = 99;
        // 插入一个写屏障，使得value = 99 强制写入主存
        StoreMemoryBarrier();
        change = true;
    }
    public void cpu1() {
        if (change) {
            // 插入一个读屏障，使得cpu1从主存获取最新数据
            LoadMemoryBarrier();
            value = 99;
        }
    }
```



总的来说，内存屏障的作用可以通过防止CPU对内存的乱序访问来保证共享数据在多线程并行执行下的可见性但是这个屏障怎么来加呢？回到最开始我们讲volatile关键字的代码，这个关键字会生成一个Lock的汇编指令，这个指令其实就相当于实现了一种内存屏障。这个时候问题又来了，内存屏障、重排序这些东西好像是和平台以及硬件架构有关系的。作为Java语言的特性，一次编写多处运行。我们不应该考虑平台相关的问题，并且这些所谓的内存屏障也不应该让程序员来关心。于是JMM的知识概念来了。



**什么是JMM**

随着CPU和内存的发展速度差异的问题，导致CPU的速度远快于内存，所以现在的CPU加入了高速缓存，高速缓存一般可以分为L1、L2、L3三级缓存。这会导致缓存一致性的问题，所以加入了**缓存一致性协议**，同时导致了内存可见性的问题，而编译器和CPU的重排序导致了原子性和有序性的问题。

JMM内存模型正是对多线程操作下的一系列规范约束，因为不可能让程序员的代码去兼容所有的CPU，通过JMM我们才屏蔽了不同硬件和操作系统内存的访问差异，这样保证了Java程序在不同的平台下达到一致的内存访问效果，同时也是保证在高效并发的时候程序能够正确执行。



![img](http://pcc.huitogo.club/656807c4e976dc42ff81ff5c0496b81b)



1. 原子性：Java内存模型通过read、load、assign、use、store、write来保证原子性操作，此外还有lock和unlock，直接对应着synchronized关键字的monitorenter和monitorexit字节码指令。
2. 可见性：可见性的问题在上面的回答已经说过，Java保证可见性可以认为通过volatile、 synchronized、final来实现。
3. 有序性：由于处理器和编译器的重排序导致的有序性问题，Java通过volatile、synchronized来保 证。