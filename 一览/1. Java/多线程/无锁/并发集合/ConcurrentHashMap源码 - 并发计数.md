**addCount(long x, int check)**

高并发计数的方法

源码如下：

```
1.  private final void addCount(long x, int check) {  

2.      CounterCell[] as;  

3.      long b, s;    

4.      // counterCells不为null的时候代表这时候是有冲突的  

5.      // 所以在尝试修改baseCount基础值失败后进入并发计数模式，也就是调用fullAddCount  

6.      if ((as = counterCells) != null || !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {  

7.          CounterCell a;  

8.          long v;  

9.          int m;  

10.         boolean uncontended = true;  

11.         if (as == null || (m = as.length - 1) < 0 || (a = as[ThreadLocalRandom.getProbe() & m]) == null  

12.                 || !(uncontended = U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {  

13.             fullAddCount(x, uncontended); // 这里跟LongAdder是一样的，可以去看LongAdder源码分析  

14.             return;   

15.         }  

16.         if (check <= 1)  

17.             return;  

18.         s = sumCount();  

19.     }  

20.     if (check >= 0) { // check是桶中的Node数量，这里判断是否要扩容，代码类似helpTransfer  

21.         Node<K, V>[] tab, nt;  

22.         int n, sc;  

23.         while (s >= (long) (sc = sizeCtl) && (tab = table) != null && (n = tab.length) < MAXIMUM_CAPACITY) {  

24.             int rs = resizeStamp(n);  

25.             if (sc < 0) {  

26.                 if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 || sc == rs + MAX_RESIZERS  

27.                         || (nt = nextTable) == null || transferIndex <= 0)  

28.                     break;  

29.                 if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))  

30.                     transfer(tab, nt);  

31.             } else if (U.compareAndSwapInt(this, SIZECTL, sc, (rs << RESIZE_STAMP_SHIFT) + 2))  

32.                 transfer(tab, null);  

33.             s = sumCount();  

34.         }  

35.     }  

36. }  
```



这里如果看了俺的LongAdder源码分析的话也是很简单的

流程步骤如下

1. 先试着去修改baseCount的值（+1），看过sumCount的源码知道concurrentHash的size就是baseCount + CounterCell[]的值，如果修改失败就是有冲突了，这时候调用fullAddCount()，其实就是Striped64.longAccumulate()，进行累积值。
2. 再判断新增后是否要扩容，走的就是helpTransfer代码，没线程就自己扩容，有线程就帮助扩容。