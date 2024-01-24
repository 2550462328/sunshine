### 1. 什么是Exchanger

首先简单介绍下Exchanger是什么？以及怎么用

故名思意，Exchanger就是一个交换器，用来**交换两个线程的数据**。



怎么使用呢？

这里已两个“玩家”私下进行装备交易来举例吧



```
1. public static void main(String[] args) { 

2.   Exchanger<String> exchanger = new Exchanger<>(); 

3.  

4.   Thread thread1 = new Thread(()->{ 

5.  

6.     String s1 = "法师权杖"; 

7.  

8.     try { 

9.       System.out.println(Thread.currentThread().getName()+" : 我要交换我的 [ " + s1 + " ]"); 

10.       s1 = exchanger.exchange(s1); 

11.  

12.       System.out.println(Thread.currentThread().getName()+" : 我交换到的装备是 [ " + s1 + " ]"); 

13.     } catch (InterruptedException ignored) { 

14.     } 

15.  

16.   },"地下道里的战士"); 

17.  

18.   Thread thread2 = new Thread(()->{ 

19.  

20.     String s2 = "战士盔甲"; 

21.  

22.     try { 

23.       System.out.println(Thread.currentThread().getName()+" : 我要交换我的 [ " + s2 + " ]"); 

24.       s2 = exchanger.exchange(s2); 

25.  

26.       System.out.println(Thread.currentThread().getName()+" : 我交换到的装备是 [ " + s2 + " ]"); 

27.     } catch (InterruptedException ignored) { 

28.     } 

29.  

30.   },"一名落魄的法师"); 

31.  

32.   thread1.start(); 

33.   thread2.start(); 

34. } 
```



通过以上的步骤，两个平民玩家在“安全交易”的前提下得到了自己想要的装备。

当然我们今天的内容是Exchanger的源码分析

先来看一下Exchanger的类UML:

![img](http://pcc.huitogo.club/a8201f568e71efbc012f5a4b0e659f6f)





Participant是什么呢？

```
1. public Exchanger() { 

2.   participant = new Participant(); 

3. } 

4.  

5. static final class Participant extends ThreadLocal<Node> { 

6.   public Node initialValue() { return new Node(); } 

7. } 
```



Participant重写了ThreadLocal的initialValue方法，也就是添加了默认值new Node

所以相当于new Exchange()的时候会在当前线程内部生成一个Node



Node是什么呢？

```
1. @sun.misc.Contended  

2. static final class Node { 

3.   int index;       // arena的下标，多个槽位的时候利用 

4.   int bound;       // 上一次记录的Exchanger.bound； 

5.   int collides;      // 在当前bound下CAS失败的次数；

6.   int hash;        // 用于自旋； 

7.   Object item;      // 这个线程的当前项，也就是需要交换的数据；

8.   volatile Object match; // 交换的数据 

9.   volatile Thread parked; // 当阻塞时，设置此线程，不阻塞的话就不必了(因为会自旋) 

10. } 
```



可以看出Node就是Exchanger实现数据交换的核心，重点是里面存放的item和match数值，记录着当前线程记录的值和期望交换的值。



和 SynchronousQueue 的区别在于， SynchronousQueue 使用了一个变量来保存数据项，通过 isData 来区别 “存” 操作和 “取” 操作。而 Exchange 使用了 2 个变量，就不用使用 isData 来区分了。



对于一个Node上交换数据我们叫单槽，同理使用多个Node交换数据叫多槽

多槽其实就是个Node数组

```
 // 多槽

1. private volatile Node[] arena; 
```



### 2. 单槽交换数据

通过追踪exchange方法，可以看到Exchanger是怎么在单槽和多槽之间选择的

```
1. public V exchange(V x) throws InterruptedException { 

2.   Object v; 

3.   Object item = (x == null) ? NULL_ITEM : x; // translate null args 

4.   if ((arena != null || 

5.     (v = slotExchange(item, false, 0L)) == null) && 

6.     ((Thread.interrupted() || // disambiguates null return 

7.      (v = arenaExchange(item, false, 0L)) == null))) 

8.     throw new InterruptedException(); 

9.   return (v == NULL_ITEM) ? null : (V)v; 

10. } 
```



具有超时功能的exchange方法

```
1. public V exchange(V x, long timeout, TimeUnit unit) 

2.   throws InterruptedException, TimeoutException { 

3.   Object v; 

4.   Object item = (x == null) ? NULL_ITEM : x; 

5.   long ns = unit.toNanos(timeout); 

6.   if ((arena != null || 

7.     (v = slotExchange(item, true, ns)) == null) && 

8.     ((Thread.interrupted() || 

9.      (v = arenaExchange(item, true, ns)) == null))) 

10.     throw new InterruptedException(); 

11.   if (v == TIMED_OUT) 

12.     throw new TimeoutException(); 

13.   return (v == NULL_ITEM) ? null : (V)v; 

14. } 
```



1. arena != nul，前面说到arena 是装Node的数组，当arena 不为空的时候自然是启用了多个Node了
2. slotExchange(item, false, 0L)== null，这个我们接下来查看slotExchange的源码会知道在单槽失败的情况下（出现了竞争）会返回null



看下slotExchange方法

```
1. private final Object slotExchange(Object item, boolean timed, long ns) { 

2.   // 得到一个初试的Node 

3.   Node p = participant.get(); 

4.   // 当前线程 

5.   Thread t = Thread.currentThread(); 

6.   // 如果发生中断，返回null,会重设中断标志位，并没有直接抛异常 

7.   if (t.isInterrupted()) // preserve interrupt status so caller can recheck 

8.     return null; 

9.  

10.   for (Node q;;) { 

11.     // 槽位 solt不为null,则说明已经有线程在这里等待交换数据了 

12.     if ((q = slot) != null) { 

13.       // 重置槽位 

14.       if (U.compareAndSwapObject(this, SLOT, q, null)) { 

15.         //获取交换的数据 

16.         Object v = q.item; 

17.         //等待线程需要的数据 

18.         q.match = item; 

19.         //等待线程 

20.         Thread w = q.parked; 

21.         //唤醒等待的线程 

22.         if (w != null) 

23.           U.unpark(w); 

24.         return v; // 返回拿到的数据，交换完成 

25.       } 

26.       // create arena on contention, but continue until slot null 

27.       //存在竞争，其它线程抢先了一步该线程，因此需要采用多槽位模式，这个后面再分析 

28.       if (NCPU > 1 && bound == 0 && 

29.         U.compareAndSwapInt(this, BOUND, 0, SEQ)) 

30.         arena = new Node[(FULL + 2) << ASHIFT]; 

31.     } 

32.     else if (arena != null) //多槽位不为空，需要执行多槽位交换 

33.       return null; // caller must reroute to arenaExchange 

34.     else { //还没有其他线程来占据槽位 

35.       p.item = item; 

36.       // 设置槽位为p(也就是槽位被当前线程占据) 

37.       if (U.compareAndSwapObject(this, SLOT, null, p)) 

38.         break; // 退出无限循环 

39.       p.item = null; // 如果设置槽位失败，则有可能其他线程抢先了，重置item,重新循环 

40.     } 

41.   } 

42.  

43.   //当前线程占据槽位，等待其它线程来交换数据 

44.   int h = p.hash; 

45.   long end = timed ? System.nanoTime() + ns : 0L; 

46.   int spins = (NCPU > 1) ? SPINS : 1; 

47.   Object v; 

48.   // 直到成功交换到数据 

49.   while ((v = p.match) == null) { 

50.     if (spins > 0) { // 自旋 

51.       h ^= h << 1; h ^= h >>> 3; h ^= h << 10; 

52.       if (h == 0) 

53.         h = SPINS | (int)t.getId(); 

54.       else if (h < 0 && (--spins & ((SPINS >>> 1) - 1)) == 0) 

55.         // 主动让出cpu,这样可以提供cpu利用率（反正当前线程也自旋等待，还不如让其它任务占用cpu） 

56.         Thread.yield();  

57.     } 

58.     else if (slot != p) //其它线程来交换数据了，修改了solt,但是还没有设置match,再稍等一会 

59.       spins = SPINS; 

60.     //需要阻塞等待其它线程来交换数据 

61.     //没发生中断，并且是单槽交换，没有设置超时或者超时时间未到 则继续执行 

62.     else if (!t.isInterrupted() && arena == null && 

63.         (!timed || (ns = end - System.nanoTime()) > 0L)) { 

64.       // cas 设置BLOCKER，可以参考Thread 中的parkBlocker 

65.       U.putObject(t, BLOCKER, this); 

66.       // 需要挂起当前线程 

67.       p.parked = t; 

68.       if (slot == p) 

69.         U.park(false, ns); // 阻塞当前线程 

70.       // 被唤醒后   

71.       p.parked = null; 

72.       // 清空 BLOCKER 

73.       U.putObject(t, BLOCKER, null); 

74.     } 

75.     // 不满足前面 else if 条件，交换失败，需要重置solt 

76.     else if (U.compareAndSwapObject(this, SLOT, p, null)) { 

77.       v = timed && ns <= 0L && !t.isInterrupted() ? TIMED_OUT : null; 

78.       break; 

79.     } 

80.   } 

81.   //清空match 

82.   U.putOrderedObject(p, MATCH, null); 

83.   p.item = null; 

84.   p.hash = h; 

85.   // 返回交换得到的数据（失败则为null） 

86.   return v; 

87. } 
```



1. 当一个线程来交换数据时，如果发现槽位（solt）有数据时，说明其它线程已经占据了槽位，等待交换数据，那么当前线程就和该槽位进行数据交换，设置相应字段，如果交换失败，则说明其它线程抢先了该线程一步和槽位交换了数据，那么这个时候就存在竞争了，这个时候就会生成多槽位（area）,后面就会进行多槽位交换了。

   

2. 如果来交换的线程发现槽位没有被占据，啊哈，这个时候自己就把槽位占据了，如果占据失败，则有可能其他线程抢先了占据了槽位，重头开始循环。

   

3. 当来交换的线程占据了槽位后，就需要等待其它线程来进行交换数据了，首先自己需要进行一定时间的自旋，因为自旋期间有可能其它线程就来了，那么这个时候就可以进行数据交换工作，而不用阻塞等待了，如果不幸，进行了一定自旋后，没有其他线程到来，那么还是避免不了需要阻塞（如果设置了超时等待，发生了超时或中断异常，则退出，不阻塞等待），当准备阻塞线程的时候，发现槽位值变了，那么说明其它线程来交换数据了，但是还没有完全准备好数据，这个时候就不阻塞了，再稍微等那么一会，如果始终没有等到其它线程来交换，那么就挂起当前线程。

   

4. 当其它线程到来并成功交换数据后，会唤醒被阻塞的线程，阻塞的线程被唤醒后，拿到数据（如果是超时，或中断，则数据为null）返回，结束。

![img](http://pcc.huitogo.club/a736276d42fc9704303ca2bd4a339383)





### 3. 多槽交换数据

多线程如果竞争激烈，那么一个槽位显然就成了性能瓶颈了，因此就衍生出了多槽位交换，各自交换各自的，互不影响。

多槽交换数是基于多槽，也就是我们之前的arena数组，那么这个数组是怎么产生的呢？



在slotExchange中有

```
1. if (NCPU > 1 && bound == 0 && 

2.           U.compareAndSwapInt(this, BOUND, 0, SEQ)) 

3.           arena = new Node[(FULL + 2) << ASHIFT]; 
```



先看一下这句话中涉及到的变量

```
1. /** 

2. * The byte distance (as a shift value) between any two used slots 

3. * in the arena. 1 << ASHIFT should be at least cacheline size. 

4. * CacheLine填充 

5. */ 

6. private static final int ASHIFT = 7; 

7.  

8. /** 

9. * The maximum supported arena index. The maximum allocatable 

10. * arena size is MMASK + 1. Must be a power of two minus one, less 

11. * than (1<<(31-ASHIFT)). The cap of 255 (0xff) more than suffices 

12. * for the expected scaling limits of the main algorithms. 

13. */ 

14. private static final int MMASK = 0xff; 

15.  

16. /** 

17. * Unit for sequence/version bits of bound field. Each successful 

18. * change to the bound also adds SEQ. 

19. * bound的"版本号" 

20. */ 

21. private static final int SEQ = MMASK + 1; 

22.  

23. /** The number of CPUs, for sizing and spin control */
private static final int NCPU = Runtime.getRuntime().availableProcessors();

24. 

25. /** 

26. * The maximum slot index of the arena: The number of slots that 

27. * can in principle hold all threads without contention, or at 

28. * most the maximum indexable value. 

29. */ 

30. static final int FULL = (NCPU >= (MMASK << 1)) ? MMASK : NCPU >>> 1; 

31.  

32. /** 

33. * Elimination array; null until enabled (within slotExchange). 

34. * Element accesses use emulation of volatile gets and CAS. 

35. */ 

36. private volatile Node[] arena; 

37.  

38. /** 

39. * The index of the largest valid arena position, OR'ed with SEQ 

40. * number in high bits, incremented on each update. The initial 

41. * update from 0 to SEQ is used to ensure that the arena array is 

42. * constructed only once. 

43. */ 

44. private volatile int bound; 
```



所以初始化arena首先判断NCPU（cpu内核数），且bound是未初始化的（防止多线程初始化arena数组），然后将bound初始化值（也叫版本号）



MMASK = 255 = 1111 1111（二进制）

SEQ = 256 = 1 0000 0000（二进制）

FULL = NCPU >= 510 ? 510 : NCPU / 2 （会有NCPU > 510个吗？超级电脑？）



所以arena数组的初始值就是 FULL + 2 ，最大可能就是NCPU /2 + 2，至于<< ASHIFT 是为了填充缓存行的



接下来正式看下arenaExchange的源码

```
1. private final Object arenaExchange(Object item, boolean timed, long ns) { 

2.   // 槽位数组 

3.   Node[] a = arena; 

4.   //代表当前线程的Node 

5.   Node p = participant.get(); // p.index 初始值为 0 

6.   for (int i = p.index;;) {           // access slot at i 

7.     int b, m, c; long j;            // j is raw array offset 

8.     //在槽位数组中根据"索引" i 取出数据 j相当于是 "第一个"槽位 

9.     Node q = (Node)U.getObjectVolatile(a, j = (i << ASHIFT) + ABASE); 

10.     // 该位置上有数据(即有线程在这里等待交换数据) 

11.     if (q != null && U.compareAndSwapObject(a, j, q, null)) { 

12.       // 进行数据交换，这里和单槽位的交换是一样的 

13.       Object v = q.item;           // release 

14.       q.match = item; 

15.       Thread w = q.parked; 

16.       if (w != null) 

17.         U.unpark(w); 

18.       return v; 

19.     } 

20.     // bound 是最大的有效的 位置，和MMASK相与，得到真正的存储数据的索引最大值 

21.     else if (i <= (m = (b = bound) & MMASK) && q == null) { 

22.       // i 在这个范围内，该槽位也为空 

23.  

24.       //将需要交换的数据 设置给p 

25.       p.item = item;             // offer 

26.       //设置该槽位数据(在该槽位等待其它线程来交换数据) 

27.       if (U.compareAndSwapObject(a, j, null, p)) { 

28.         long end = (timed && m == 0) ? System.nanoTime() + ns : 0L; 

29.         Thread t = Thread.currentThread(); // wait 

30.         // 进行一定时间的自旋 

31.         for (int h = p.hash, spins = SPINS;;) { 

32.           Object v = p.match; 

33.           //在自旋的过程中，有线程来和该线程交换数据 

34.           if (v != null) { 

35.             //交换数据后，清空部分设置，返回交换得到的数据，over 

36.             U.putOrderedObject(p, MATCH, null); 

37.             p.item = null;       // clear for next use 

38.             p.hash = h; 

39.             return v; 

40.           } 

41.           else if (spins > 0) { 

42.             h ^= h << 1; h ^= h >>> 3; h ^= h << 10; // xorshift 

43.             if (h == 0)        // initialize hash 

44.               h = SPINS | (int)t.getId(); 

45.             else if (h < 0 &&     // approx 50% true 

46.                 (--spins & ((SPINS >>> 1) - 1)) == 0) 

47.               Thread.yield();    // two yields per wait 

48.           } 

49.           // 交换数据的线程到来，但是还没有设置好match，再稍等一会 

50.           else if (U.getObjectVolatile(a, j) != p) 

51.             spins = SPINS;  

52.           //符合条件，特别注意m==0 这个说明已经到达area 中最小的存储数据槽位了 

53.           //没有其他线程在槽位等待了，所有当前线程需要阻塞在这里    

54.           else if (!t.isInterrupted() && m == 0 && 

55.               (!timed || 

56.                (ns = end - System.nanoTime()) > 0L)) { 

57.             U.putObject(t, BLOCKER, this); // emulate LockSupport 

58.             p.parked = t;       // minimize window 

59.             // 再次检查槽位，看看在阻塞前，有没有线程来交换数据 

60.             if (U.getObjectVolatile(a, j) == p)  

61.               U.park(false, ns); // 挂起 

62.             p.parked = null; 

63.             U.putObject(t, BLOCKER, null); 

64.           } 

65.           // 当前这个槽位一直没有线程来交换数据，准备换个槽位试试 

66.           else if (U.getObjectVolatile(a, j) == p && 

67.               U.compareAndSwapObject(a, j, p, null)) { 

68.             //更新bound 

69.             if (m != 0)        // try to shrink 

70.               U.compareAndSwapInt(this, BOUND, b, b + SEQ - 1); 

71.             p.item = null; 

72.             p.hash = h; 

73.             // 减小索引值 往"第一个"槽位的方向挪动 

74.             i = p.index >>>= 1;    // descend 

75.             // 发送中断，返回null 

76.             if (Thread.interrupted()) 

77.               return null; 

78.             // 超时 

79.             if (timed && m == 0 && ns <= 0L) 

80.               return TIMED_OUT; 

81.             break;           // expired; restart 继续主循环 

82.           } 

83.         } 

84.       } 

85.       else 

86.         //占据槽位失败，先清空item,防止成功交换数据后，p.item还引用着item 

87.         p.item = null;           // clear offer 

88.     } 

89.     else { // i 不在有效范围，或者被其它线程抢先了 

90.       //更新p.bound 

91.       if (p.bound != b) {          // stale; reset 

92.         p.bound = b; 

93.         //新bound ，重置collides 

94.         p.collides = 0; 

95.         //i如果达到了最大，那么就递减 

96.         i = (i != m || m == 0) ? m : m - 1; 

97.       } 

98.       else if ((c = p.collides) < m || m == FULL || 

99.           !U.compareAndSwapInt(this, BOUND, b, b + SEQ + 1)) { 

100.         p.collides = c + 1; // 更新冲突 

101.         // i=0 那么就从m开始，否则递减i 

102.         i = (i == 0) ? m : i - 1;     // cyclically traverse 

103.       } 

104.       else 

105.         //递增，往后挪动 

106.         i = m + 1;             // grow 

107.       // 更新index 

108.       p.index = i; 

109.     } 

110.   } 

111. } 
```



可能不对（后述进一步查证修改）



多槽的交换大致思想就是：当一个线程来交换的时候，如果”第一个”槽位是空的，那么自己就在那里等待，如果发现”第一个”槽位有等待线程，那么就直接交换，如果交换失败，说明其它线程在进行交换，那么就往后挪一个槽位，如果有数据就交换，没数据就等一会，但是不会阻塞在这里，在这里等了一会，发现还没有其它线程来交换数据，那么就往“第一个”槽位的方向挪，如果反复这样过后，挪到了第一个槽位，没有线程来交换数据了，那么自己就在”第一个”槽位阻塞等待。



简单来说，如果有竞争冲突，那么就寻找后面的槽位，在后面的槽位等待一定时间，没有线程来交换，那么就又往前挪。



### 4. 总结

**1 ）exchanger底层是CAS操作 + unsafe的park和unpark**

**2）exchanger是分单槽和多槽的情况的**

1. 在没有竞争的情况下，多线程在单槽上完成数据交换，线程A进入exchange，如果单槽为空，将自身Node放入槽中并unsafe.park；线程B进入exchange，如果单槽不为空，交换数据，并唤醒线程A
2. 在有竞争的情况下，有竞争发生在线程B进入槽中发现单槽不为空，开开心心的准备交换自己的数据之前先将槽子置为空，但是这个过程失败了，说明槽中的数据已经被人取走了，只能走多槽模式
3. 在多槽模式下，会有一个index记录槽的下标，index默认为0，如果槽0有竞争，那么线程B会去槽1进行交换或者等待交换，在多次失败或者其他情况后，index会往回走，也就是往槽0靠。