首先我们看下**Redis单线程为什么还能这么快？**

1. 命令执行基于内存操作，一条命令在内存里操作的时间是几十纳秒
2. 令执行是单线程操作，没有线程切换开销。
3. 基于IO多路复用机制提升Redis的1/O利用率。
4. 高效的数据存储结构:全局hash表以及多种高效数据结构，比如:**跳表**，压缩列表，链表等等。



今天看一下跳表为什么会这么好用



#### 1. 跳跃表的概念

跳跃表（skiplist）是一种随机化的数据结构，是一种可以与平衡树媲美的层次化链表结构——查找、删除、添加等操作都可以在对数期望时间下完成，以下是一个典型的跳跃表例子：

![img](http://pcc.huitogo.club/f925084097dff5d8313eb47fefb61278)



我们在上一篇中提到了 Redis 的五种基本结构中，有一个叫做 有序列表 zset 的数据结构，它类似于 Java 中的 SortedSet 和 HashMap 的结合体，一方面它是一个 set 保证了内部 value 的唯一性，另一方面又可以给每个 value 赋予一个排序的权重值 score，来达到 排序 的目的。



它的内部实现就依赖了一种叫做 「跳跃列表」 的数据结构。



**为什么使用跳跃表？**

首先，因为 zset 要支持随机的插入和删除，所以它 不宜使用数组来实现，关于排序问题，我们也很容易就想到 红黑树/ 平衡树 这样的树形结构，为什么 Redis 不使用这样一些结构呢？

- **性能考虑**： 在高并发的情况下，树形结构需要执行一些类似于 rebalance 这样的可能涉及整棵树的操作，相对来说跳跃表的变化只涉及局部 (下面详细说)；
- **实现考虑**： 在复杂度与红黑树相同的情况下，跳跃表实现起来更简单，看起来也更加直观；



基于以上的一些考虑，Redis 基于 William Pugh 的论文做出一些改进后采用了 跳跃表 这样的结构。



本质是**解决查找问题**



我们先来看一个普通的链表结构：

![img](http://pcc.huitogo.club/236fe0208c87ddc87720e26c914b685c)



我们需要这个链表按照 score 值进行排序，这也就意味着，当我们需要添加新的元素时，我们需要定位到插入点，这样才可以继续保证链表是有序的，通常我们会使用 二分查找法，但二分查找是有序数组的，链表没办法进行位置定位，我们除了遍历整个找到第一个比给定数据大的节点为止 （时间复杂度 O(n)) 似乎没有更好的办法。



但假如我们每相邻两个节点之间就增加一个指针，让指针指向下一个节点，如下图：

![img](http://pcc.huitogo.club/9b01f63888653d15f4374efb9a862a22)



这样所有新增的指针连成了一个新的链表，但它包含的数据却只有原来的一半（图中的为 3，11）。



现在假设我们想要查找数据时，可以根据这条新的链表查找，如果碰到比待查找数据大的节点时，再回到原来的链表中进行查找，比如，我们想要查找 7，查找的路径则是沿着下图中标注出的红色指针所指向的方向进行的：

![img](http://pcc.huitogo.club/bb01575e2d99ec7a832a6666022c2405)



这是一个略微极端的例子，但我们仍然可以看到，通过新增加的指针查找，我们不再需要与链表上的每一个节点逐一进行比较，这样改进之后需要比较的节点数大概只有原来的一半。



利用同样的方式，我们可以在新产生的链表上，继续为每两个相邻的节点增加一个指针，从而产生第三层链表：

![img](http://pcc.huitogo.club/41e6630561a2907839860a961ac7a002)



在这个新的三层链表结构中，我们试着 查找 13，那么沿着最上层链表首先比较的是 11，发现 11 比 13 小，于是我们就知道只需要到 11 后面继续查找，从而一下子跳过了 11 前面的所有节点。



可以想象，当链表足够长，这样的多层链表结构可以帮助我们跳过很多下层节点，从而加快查找的效率。



**更进一步的跳跃表**

跳跃表 skiplist 就是受到这种多层链表结构的启发而设计出来的。按照上面生成链表的方式，上面每一层链表的节点个数，是下面一层的节点个数的一半，这样查找过程就非常类似于一个二分查找，使得查找的时间复杂度可以降低到 O(logn)。



但是，这种方法在插入数据的时候有很大的问题。新插入一个节点之后，就会打乱上下相邻两层链表上节点个数严格的 2:1 的对应关系。如果要维持这种对应关系，就必须把新插入的节点后面的所有节点 （也包括新插入的节点） 重新进行调整，这会让时间复杂度重新蜕化成 *O(n)*。删除数据也有同样的问题。



skiplist 为了避免这一问题，它不要求上下相邻两层链表之间的节点个数有严格的对应关系，而是 为每个节点随机出一个层数(level)。比如，一个节点随机出的层数是 3，那么就把它链入到第 1 层到第 3 层这三层链表中。为了表达清楚，下图展示了如何通过一步步的插入操作从而形成一个 skiplist 的过程：

![img](http://pcc.huitogo.club/0988f9a43292221a16da5cbcec38e3ce)



从上面的创建和插入的过程中可以看出，每一个节点的层数（level）是随机出来的，而且新插入一个节点并不会影响到其他节点的层数，因此，**插入操作只需要修改节点前后的指针，而不需要对多个节点都进行调整**，这就降低了插入操作的复杂度。



现在我们假设从我们刚才创建的这个结构中查找 23 这个不存在的数，那么查找路径会如下图：

![img](http://pcc.huitogo.club/2bb1addd3767c7166d01d91b0e9d384c)



#### 2. redis中skiplist的实现

Redis 中的跳跃表由 server.h/zskiplistNode 和 server.h/zskiplist 两个结构定义，前者为跳跃表节点，后者则保存了跳跃节点的相关信息，同之前的 集合 list 结构类似，其实只有 zskiplistNode 就可以实现了，但是引入后者是为了更加方便的操作：



##### 2.1 zskiplistNode节点

```
1. /* ZSETs use a specialized version of Skiplists */ 

2. typedef struct zskiplistNode { 

3.   // value 

4.   sds ele; 

5.   // 分值 

6.   double score; 

7.   // 后退指针 

8.   struct zskiplistNode *backward; 

9.   // 层 

10.   struct zskiplistLevel { 

11.     // 前进指针 

12.     struct zskiplistNode *forward; 

13.     // 跨度 

14.     unsigned long span; 

15.   } level[]; 

16. } zskiplistNode; 

17.  

18. typedef struct zskiplist { 

19.   // 跳跃表头指针 

20.   struct zskiplistNode *header, *tail; 

21.   // 表中节点的数量 

22.   unsigned long length; 

23.   // 表中层数最大的节点的层数 

24.   int level; 

25. } zskiplist; 
```



##### 2.2 随机层数算法

```
1. int zslRandomLevel(void) { 

2.   int level = 1; 

3.   while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF)) 

4.     level += 1; 

5.   return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL; 

6. } 
```



直观上期望的目标是 50% 的概率被分配到 Level 1，25% 的概率被分配到 Level 2，12.5% 的概率被分配到 Level 3，以此类推…有 2-63 的概率被分配到最顶层，因为这里每一层的晋升率都是 50%。



**Redis 跳跃表默认允许最大的层数是 32**，被源码中 ZSKIPLIST_MAXLEVEL 定义，当 Level[0] 有 264 个元素时，才能达到 32 层，所以定义 32 完全够用了。



##### 2.3 初始化跳跃表

```
1. zskiplist *zslCreate(void) { 

2.   int j; 

3.   zskiplist *zsl; 

4.  

5.   // 申请内存空间 

6.   zsl = zmalloc(sizeof(*zsl)); 

7.   // 初始化层数为 1 

8.   zsl->level = 1; 

9.   // 初始化长度为 0 

10.   zsl->length = 0; 

11.   // 创建一个层数为 32，分数为 0，没有 value 值的跳跃表头节点 

12.   zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL); 

13.  

14.   // 跳跃表头节点初始化 

15.   for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) { 

16.     // 将跳跃表头节点的所有前进指针 forward 设置为 NULL 

17.     zsl->header->level[j].forward = NULL; 

18.     // 将跳跃表头节点的所有跨度 span 设置为 0 

19.     zsl->header->level[j].span = 0; 

20.   } 

21.   // 跳跃表头节点的后退指针 backward 置为 NULL 

22.   zsl->header->backward = NULL; 

23.   // 表头指向跳跃表尾节点的指针置为 NULL 

24.   zsl->tail = NULL; 

25.   return zsl; 

26. } 
```



即执行完之后创建了如下结构的初始化跳跃表：

![img](http://pcc.huitogo.club/d25285f496183da6c5d02ed57c8fc2e3)



##### 2.4 插入节点实现

redis实现跳跃表核心代码，它需要做两件事：

1. 找到当前我需要插入的位置 （其中包括相同 score 时的处理）；
2. 创建新节点，调整前后的指针指向，完成插入；



具体实现步骤如下：



第一部分：声明需要存储的变量

```
1. // 存储搜索路径 

2. zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x; 

3. // 存储经过的节点跨度 

4. unsigned int rank[ZSKIPLIST_MAXLEVEL]; 

5. int i, level; 
```



第二部分：搜索当前节点插入位置

```
1. serverAssert(!isnan(score)); 

2. x = zsl->header; 

3. // 逐步降级寻找目标节点，得到 "搜索路径" 

4. for (i = zsl->level-1; i >= 0; i--) { 

5.   /* store rank that is crossed to reach the insert position */ 

6.   rank[i] = i == (zsl->level-1) ? 0 : rank[i+1]; 

7.   // 如果 score 相等，还需要比较 value 值 

8.   while (x->level[i].forward && 

9.       (x->level[i].forward->score < score || 

10.         (x->level[i].forward->score == score && 

11.         sdscmp(x->level[i].forward->ele,ele) < 0))) 

12.   { 

13.     rank[i] += x->level[i].span; 

14.     x = x->level[i].forward; 

15.   } 

16.   // 记录 "搜索路径" 

17.   update[i] = x; 

18. } 
```



讨论： 有一种极端的情况，就是跳跃表中的所有 score 值都是一样，zset 的查找性能会不会退化为 O(n) 呢？



从上面的源码中我们可以发现 zset 的排序元素不只是看 score 值，也会比较 value 值 （字符串比较）



第三部分：生成插入节点

```
1. /* we assume the element is not already inside, since we allow duplicated 

2. * scores, reinserting the same element should never happen since the 

3. * caller of zslInsert() should test in the hash table if the element is 

4. * already inside or not. */ 

5. level = zslRandomLevel(); 

6. // 如果随机生成的 level 超过了当前最大 level 需要更新跳跃表的信息 

7. if (level > zsl->level) { 

8.   for (i = zsl->level; i < level; i++) { 

9.     rank[i] = 0; 

10.     update[i] = zsl->header; 

11.     update[i]->level[i].span = zsl->length; 

12.   } 

13.   zsl->level = level; 

14. } 

15. // 创建新节点 

16. x = zslCreateNode(level,score,ele); 
```



第四部分：重排前向指针

```
1. for (i = 0; i < level; i++) { 

2.   x->level[i].forward = update[i]->level[i].forward; 

3.   update[i]->level[i].forward = x; 

4.  

5.   /* update span covered by update[i] as x is inserted here */ 

6.   x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]); 

7.   update[i]->level[i].span = (rank[0] - rank[i]) + 1; 

8. } 

9.  

10. /* increment span for untouched levels */ 

11. for (i = level; i < zsl->level; i++) { 

12.   update[i]->level[i].span++; 

13. } 
```



第五部分：重排后向指针并返回

```
1. x->backward = (update[0] == zsl->header) ? NULL : update[0]; 

2. if (x->level[0].forward) 

3.   x->level[0].forward->backward = x; 

4. else 

5.   zsl->tail = x; 

6. zsl->length++; 

7. return x; 
```



##### 2.5 删除节点实现

删除过程由源码中的 t_zset.c/zslDeleteNode 定义，和插入过程类似，都需要先把这个 “搜索路径” 找出来，然后对于每个层的相关节点重排一下前向后向指针，同时还要注意更新一下最高层数 maxLevel，直接放源码

```
1. /* Internal function used by zslDelete, zslDeleteByScore and zslDeleteByRank */ 

2. void zslDeleteNode(zskiplist *zsl, zskiplistNode *x, zskiplistNode **update) { 

3.   int i; 

4.   for (i = 0; i < zsl->level; i++) { 

5.     if (update[i]->level[i].forward == x) { 

6.       update[i]->level[i].span += x->level[i].span - 1; 

7.       update[i]->level[i].forward = x->level[i].forward; 

8.     } else { 

9.       update[i]->level[i].span -= 1; 

10.     } 

11.   } 

12.   if (x->level[0].forward) { 

13.     x->level[0].forward->backward = x->backward; 

14.   } else { 

15.     zsl->tail = x->backward; 

16.   } 

17.   while(zsl->level > 1 && zsl->header->level[zsl->level-1].forward == NULL) 

18.     zsl->level--; 

19.   zsl->length--; 

20. } 

21.  

22. /* Delete an element with matching score/element from the skiplist. 

23. * The function returns 1 if the node was found and deleted, otherwise 

24. * 0 is returned. 

25. * 

26. * If 'node' is NULL the deleted node is freed by zslFreeNode(), otherwise 

27. * it is not freed (but just unlinked) and *node is set to the node pointer, 

28. * so that it is possible for the caller to reuse the node (including the 

29. * referenced SDS string at node->ele). */ 

30. int zslDelete(zskiplist *zsl, double score, sds ele, zskiplistNode **node) { 

31.   zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x; 

32.   int i; 

33.  

34.   x = zsl->header; 

35.   for (i = zsl->level-1; i >= 0; i--) { 

36.     while (x->level[i].forward && 

37.         (x->level[i].forward->score < score || 

38.           (x->level[i].forward->score == score && 

39.           sdscmp(x->level[i].forward->ele,ele) < 0))) 

40.     { 

41.       x = x->level[i].forward; 

42.     } 

43.     update[i] = x; 

44.   } 

45.   /* We may have multiple elements with the same score, what we need 

46.   * is to find the element with both the right score and object. */ 

47.   x = x->level[0].forward; 

48.   if (x && score == x->score && sdscmp(x->ele,ele) == 0) { 

49.     zslDeleteNode(zsl, x, update); 

50.     if (!node) 

51.       zslFreeNode(x); 

52.     else 

53.       *node = x; 

54.     return 1; 

55.   } 

56.   return 0; /* not found */ 

57. } 
```



##### 2.6 更新节点实现

当我们调用 ZADD 方法时，如果对应的 value 不存在，那就是插入过程，如果这个 value 已经存在，只是调整一下 score 的值，那就需要走一个更新流程。



假设这个新的 score 值并不会带来排序上的变化，那么就不需要调整位置，直接修改元素的 score 值就可以了，但是如果排序位置改变了，那就需要调整位置，该如何调整呢？



从源码 t_zset.c/zsetAdd 函数 1350 行左右可以看到，Redis 采用了一个非常简单的策略：

```
1. /* Remove and re-insert when score changed. */ 

2. if (score != curscore) { 

3.   zobj->ptr = zzlDelete(zobj->ptr,eptr); 

4.   zobj->ptr = zzlInsert(zobj->ptr,ele,score); 

5.   *flags |= ZADD_UPDATED; 

6. } 
```



**把这个元素删除再插入这个**，需要经过两次路径搜索，从这一点上来看，Redis 的 ZADD 代码似乎还有进一步优化的空间。



##### 2.7 元素排名的实现

跳跃表本身是有序的，Redis 在 skiplist 的 forward 指针上进行了优化，给每一个 forward 指针都增加了 span 属性，用来 表示从前一个节点沿着当前层的 forward 指针跳到当前这个节点中间会跳过多少个节点。在上面的源码中我们也可以看到 Redis 在插入、删除操作时都会小心翼翼地更新 span 值的大小。



所以，沿着 “搜索路径”，把所有经过节点的跨度 span 值进行累加就可以算出当前元素的最终 rank 值了：

```
1. /* Find the rank for an element by both score and key. 

2. * Returns 0 when the element cannot be found, rank otherwise. 

3. * Note that the rank is 1-based due to the span of zsl->header to the 

4. * first element. */ 

5. unsigned long zslGetRank(zskiplist *zsl, double score, sds ele) { 

6.   zskiplistNode *x; 

7.   unsigned long rank = 0; 

8.   int i; 

9.  

10.   x = zsl->header; 

11.   for (i = zsl->level-1; i >= 0; i--) { 

12.     while (x->level[i].forward && 

13.       (x->level[i].forward->score < score || 

14.         (x->level[i].forward->score == score && 

15.         sdscmp(x->level[i].forward->ele,ele) <= 0))) { 

16.       // span 累加 

17.       rank += x->level[i].span; 

18.       x = x->level[i].forward; 

19.     } 

20.  

21.     /* x might be equal to zsl->header, so test if obj is non-NULL */ 

22.     if (x->ele && sdscmp(x->ele,ele) == 0) { 

23.       return rank; 

24.     } 

25.   } 

26.   return 0; 

27. } 
```