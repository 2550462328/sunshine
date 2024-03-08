#### 1. Stream持久化的发布/订阅系统

Redis Stream 从概念上来说，就像是一个 仅追加内容 的 消息链表，把所有加入的消息都一个一个串起来，每个消息都有一个唯一的 ID 和内容，这很简单，让它复杂的是从 Kafka 借鉴的另一种概念：消费者组(Consumer Group) (思路一致，实现不同)：

![img](http://pcc.huitogo.club/90c33fd0c268e889c0caf8bcec3ba3cb)



上图就展示了一个典型的 Stream 结构。每个 Stream 都有唯一的名称，它就是 Redis 的 key，在我们首次使用 xadd 指令追加消息时自动创建。我们对图中的一些概念做一下解释：



**1）Consumer Group**

消费者组，可以简单看成记录流状态的一种数据结构。消费者既可以选择使用 XREAD 命令进行 **独立消费**，也可以多个消费者同时加入一个消费者组进行 **组内消费**。**同一个消费者组内的消费者共享所有的 Stream 信息，同一条消息只会有一个消费者消费到**，这样就可以应用在分布式的应用场景中来保证消息的唯一性。



**2）last_delivered_id**

用来表示消费者组消费在 Stream 上 **消费位置** 的游标信息。每个消费者组都有一个 Stream 内 唯一的名称，消费者组不会自动创建，需要使用 XGROUP CREATE 指令来显式创建，并且需要指定从哪一个消息 ID 开始消费，用来初始化 last_delivered_id 这个变量。



**3）pending_ids**

每个消费者内部都有的一个状态变量，**用来表示 已经 被客户端 获取，但是 还没有 ack 的消息**。记录的目的是为了 **保证客户端至少消费了消息一次**，而不会在网络传输的中途丢失而没有对消息进行处理。如果客户端没有 ack，那么这个变量里面的消息 ID 就会越来越多，一旦某个消息被 ack，它就会对应开始减少。这个变量也被 Redis 官方称为 PEL (Pending Entries List)。



**4）消息Id**

消息 ID 如果是由 XADD 命令返回自动创建的话，那么它的格式会像这样：**timestampInMillis-sequence** (毫秒时间戳-序列号)，例如 1527846880585-5，它表示当前的消息是在毫秒时间戳 1527846880585 时产生的，并且是该毫秒内产生的第 5 条消息。



这些 ID 的格式看起来有一些奇怪，为什么要使用时间来当做 ID 的一部分呢？ 一方面，我们要 **满足 ID 自增** 的属性，另一方面，也是为了 **支持范围查找** 的功能。由于 ID 和生成消息的时间有关，这样就使得在根据时间范围内查找时基本上是没有额外损耗的。



当然消息 ID 也可以由客户端自定义，但是形式必须是 “整数-整数”，而且后面加入的消息的 ID 必须要大于前面的消息 ID。



**5）消息内容**

消息内容就是普通的键值对，形如 hash 结构的键值对。



##### 1.1 Stream的使用

**首先是对Stream这个数据结构的增删改查**

- xadd：追加消息
- xdel：删除消息，这里的删除仅仅是设置了标志位，不影响消息总长度
- xrange：获取消息列表，会自动过滤已经删除的消息
- xlen：消息长度
- del：删除Stream



使用示例如下：

```
1. # *号表示服务器自动生成ID，后面顺序跟着一堆key/value 

2. 127.0.0.1:6379> xadd codehole * name laoqian age 30 # 名字叫laoqian，年龄30岁 

3. 1527849609889-0 # 生成的消息ID 

4. 127.0.0.1:6379> xadd codehole * name xiaoyu age 29 

5. 1527849629172-0 

6. 127.0.0.1:6379> xadd codehole * name xiaoqian age 1 

7. 1527849637634-0 

8. 127.0.0.1:6379> xlen codehole 

9. (integer) 3 

10. 127.0.0.1:6379> xrange codehole - + # -表示最小值, +表示最大值 

11. 1) 1) 1527849609889-0 

12.  2) 1) "name" 

13.    2) "laoqian" 

14.    3) "age" 

15.    4) "30" 

16. 2) 1) 1527849629172-0 

17.  2) 1) "name" 

18.    2) "xiaoyu" 

19.    3) "age" 

20.    4) "29" 

21. 3) 1) 1527849637634-0 

22.  2) 1) "name" 

23.    2) "xiaoqian" 

24.    3) "age" 

25.    4) "1" 

26. 127.0.0.1:6379> xrange codehole 1527849629172-0 + # 指定最小消息ID的列表 

27. 1) 1) 1527849629172-0 

28.  2) 1) "name" 

29.    2) "xiaoyu" 

30.    3) "age" 

31.    4) "29" 

32. 2) 1) 1527849637634-0 

33.  2) 1) "name" 

34.    2) "xiaoqian" 

35.    3) "age" 

36.    4) "1" 

37. 127.0.0.1:6379> xrange codehole - 1527849629172-0 # 指定最大消息ID的列表 

38. 1) 1) 1527849609889-0 

39.  2) 1) "name" 

40.    2) "laoqian" 

41.    3) "age" 

42.    4) "30" 

43. 2) 1) 1527849629172-0 

44.  2) 1) "name" 

45.    2) "xiaoyu" 

46.    3) "age" 

47.    4) "29" 

48. 127.0.0.1:6379> xdel codehole 1527849609889-0 

49. (integer) 1 

50. 127.0.0.1:6379> xlen codehole # 长度不受影响 

51. (integer) 3 

52. 127.0.0.1:6379> xrange codehole - + # 被删除的消息没了 

53. 1) 1) 1527849629172-0 

54.  2) 1) "name" 

55.    2) "xiaoyu" 

56.    3) "age" 

57.    4) "29" 

58. 2) 1) 1527849637634-0 

59.  2) 1) "name" 

60.    2) "xiaoqian" 

61.    3) "age" 

62.    4) "1" 

63. 127.0.0.1:6379> del codehole # 删除整个Stream 

64. (integer) 1 
```



##### 1.2 对于Stream的独立消费

我们可以在不定义消费组的情况下进行 Stream 消息的 独立消费，当 Stream 没有新消息时，甚至可以阻塞等待。Redis 设计了一个单独的消费指令 xread，可以将 Stream 当成普通的消息队列(list)来使用。使用 xread 时，我们可以完全忽略 消费组(Consumer Group) 的存在，就好比 Stream 就是一个普通的列表(list)：

```
1. # 从Stream头部读取两条消息 

2. 127.0.0.1:6379> xread count 2 streams codehole 0-0 

3. 1) 1) "codehole" 

4.  2) 1) 1) 1527851486781-0 

5.     2) 1) "name" 

6.       2) "laoqian" 

7.       3) "age" 

8.       4) "30" 

9.    2) 1) 1527851493405-0 

10.     2) 1) "name" 

11.       2) "yurui" 

12.       3) "age" 

13.       4) "29" 

14. # 从Stream尾部读取一条消息，毫无疑问，这里不会返回任何消息 

15. 127.0.0.1:6379> xread count 1 streams codehole $ 

16. (nil) 

17. # 从尾部阻塞等待新消息到来，下面的指令会堵住，直到新消息到来 

18. 127.0.0.1:6379> xread block 0 count 1 streams codehole $ 

19. # 我们从新打开一个窗口，在这个窗口往Stream里塞消息 

20. 127.0.0.1:6379> xadd codehole * name youming age 60 

21. 1527852774092-0 

22. # 再切换到前面的窗口，我们可以看到阻塞解除了，返回了新的消息内容 

23. # 而且还显示了一个等待时间，这里我们等待了93s 

24. 127.0.0.1:6379> xread block 0 count 1 streams codehole $ 

25. 1) 1) "codehole" 

26.  2) 1) 1) 1527852774092-0 

27.     2) 1) "name" 

28.       2) "youming" 

29.       3) "age" 

30.       4) "60" 

31. (93.11s) 
```



客户端如果想要使用 xread 进行 顺序消费，一定要 记住当前消费 到哪里了，也就是返回的消息 ID。下次继续调用 xread 时，将上次返回的最后一个消息 ID 作为参数传递进去，就可以继续消费后续的消息。



**block 0 表示永远阻塞，直到消息到来，block 1000 表示阻塞 1s，如果 1s 内没有任何消息到来，就返回 nil：**

```
1. 127.0.0.1:6379> xread block 1000 count 1 streams codehole $ 

2. (nil) 

3. (1.07s) 
```



##### 1.3 对于Stream的组内消费

Stream 通过 xgroup create 指令创建消费组(Consumer Group)，需要传递起始消息 ID 参数用来初始化 last_delivered_id 变量：

```
1. 127.0.0.1:6379> xgroup create codehole cg1 0-0 # 表示从头开始消费 

2. OK 

3. # $表示从尾部开始消费，只接受新消息，当前Stream消息会全部忽略 

4. 127.0.0.1:6379> xgroup create codehole cg2 $ 

5. OK 

6. 127.0.0.1:6379> xinfo codehole # 获取Stream信息 

7. 1) length 

8. 2) (integer) 3 # 共3个消息 

9. 3) radix-tree-keys 

10. 4) (integer) 1 

11. 5) radix-tree-nodes 

12. 6) (integer) 2 

13. 7) groups 

14. 8) (integer) 2 # 两个消费组 

15. 9) first-entry # 第一个消息 

16. 10) 1) 1527851486781-0 

17.   2) 1) "name" 

18.    2) "laoqian" 

19.    3) "age" 

20.    4) "30" 

21. 11) last-entry # 最后一个消息 

22. 12) 1) 1527851498956-0 

23.   2) 1) "name" 

24.    2) "xiaoqian" 

25.    3) "age" 

26.    4) "1" 

27. 127.0.0.1:6379> xinfo groups codehole # 获取Stream的消费组信息 

28. 1) 1) name 

29.  2) "cg1" 

30.  3) consumers 

31.  4) (integer) 0 # 该消费组还没有消费者 

32.  5) pending 

33.  6) (integer) 0 # 该消费组没有正在处理的消息 

34. 2) 1) name 

35.  2) "cg2" 

36.  3) consumers # 该消费组还没有消费者 

37.  4) (integer) 0 

38.  5) pending 

39.  6) (integer) 0 # 该消费组没有正在处理的消息 
```



Stream 提供了 xreadgroup 指令可以进行消费组的组内消费，需要提供 **消费组名称、消费者名称和起始消息 ID**。它同 xread 一样，也可以阻塞等待新消息。读到新消息后，对应的消息 ID 就会进入消费者的 PEL (正在处理的消息) 结构里，**客户端处理完毕后使用 xack 指令 通知服务器**，本条消息已经处理完毕，该消息 **ID 就会从 PEL 中移除**



下面是示例：

```
1. # >号表示从当前消费组的last_delivered_id后面开始读 

2. # 每当消费者读取一条消息，last_delivered_id变量就会前进 

3. 127.0.0.1:6379> xreadgroup GROUP cg1 c1 count 1 streams codehole > 

4. 1) 1) "codehole" 

5.  2) 1) 1) 1527851486781-0 

6.     2) 1) "name" 

7.       2) "laoqian" 

8.       3) "age" 

9.       4) "30" 

10. 127.0.0.1:6379> xreadgroup GROUP cg1 c1 count 1 streams codehole > 

11. 1) 1) "codehole" 

12.  2) 1) 1) 1527851493405-0 

13.     2) 1) "name" 

14.       2) "yurui" 

15.       3) "age" 

16.       4) "29" 

17. 127.0.0.1:6379> xreadgroup GROUP cg1 c1 count 2 streams codehole > 

18. 1) 1) "codehole" 

19.  2) 1) 1) 1527851498956-0 

20.     2) 1) "name" 

21.       2) "xiaoqian" 

22.       3) "age" 

23.       4) "1" 

24.    2) 1) 1527852774092-0 

25.     2) 1) "name" 

26.       2) "youming" 

27.       3) "age" 

28.       4) "60" 

29. # 再继续读取，就没有新消息了 

30. 127.0.0.1:6379> xreadgroup GROUP cg1 c1 count 1 streams codehole > 

31. (nil) 

32. # 那就阻塞等待吧 

33. 127.0.0.1:6379> xreadgroup GROUP cg1 c1 block 0 count 1 streams codehole > 

34. # 开启另一个窗口，往里塞消息 

35. 127.0.0.1:6379> xadd codehole * name lanying age 61 

36. 1527854062442-0 

37. # 回到前一个窗口，发现阻塞解除，收到新消息了 

38. 127.0.0.1:6379> xreadgroup GROUP cg1 c1 block 0 count 1 streams codehole > 

39. 1) 1) "codehole" 

40.  2) 1) 1) 1527854062442-0 

41.     2) 1) "name" 

42.       2) "lanying" 

43.       3) "age" 

44.       4) "61" 

45. (36.54s) 

46. 127.0.0.1:6379> xinfo groups codehole # 观察消费组信息 

47. 1) 1) name 

48.  2) "cg1" 

49.  3) consumers 

50.  4) (integer) 1 # 一个消费者 

51.  5) pending 

52.  6) (integer) 5 # 共5条正在处理的信息还有没有ack 

53. 2) 1) name 

54.  2) "cg2" 

55.  3) consumers 

56.  4) (integer) 0 # 消费组cg2没有任何变化，因为前面我们一直在操纵cg1 

57.  5) pending 

58.  6) (integer) 0 

59. # 如果同一个消费组有多个消费者，我们可以通过xinfo consumers指令观察每个消费者的状态 

60. 127.0.0.1:6379> xinfo consumers codehole cg1 # 目前还有1个消费者 

61. 1) 1) name 

62.  2) "c1" 

63.  3) pending 

64.  4) (integer) 5 # 共5条待处理消息 

65.  5) idle 

66.  6) (integer) 418715 # 空闲了多长时间ms没有读取消息了 

67. # 接下来我们ack一条消息 

68. 127.0.0.1:6379> xack codehole cg1 1527851486781-0 

69. (integer) 1 

70. 127.0.0.1:6379> xinfo consumers codehole cg1 

71. 1) 1) name 

72.  2) "c1" 

73.  3) pending 

74.  4) (integer) 4 # 变成了5条 

75.  5) idle 

76.  6) (integer) 668504 

77. # 下面ack所有消息 

78. 127.0.0.1:6379> xack codehole cg1 1527851493405-0 1527851498956-0 1527852774092-0 1527854062442-0 

79. (integer) 4 

80. 127.0.0.1:6379> xinfo consumers codehole cg1 

81. 1) 1) name 

82.  2) "c1" 

83.  3) pending 

84.  4) (integer) 0 # pel空了 

85.  5) idle 

86.  6) (integer) 745505 
```



#### 2. 关于Stream在实际使用中的一些考虑

##### 2.1 Stream的内存

可以看出一个Stream数据结构在实际使用中数量是一直增长的，消息被消费了也只是修改下消费位，那这个数量有没有上限呢？内存方面怎么处理的？

Redis提供了一个定长 Stream 功能。在 xadd 的指令提供一个定长长度 maxlen，就可以将老的消息干掉，确保最多不超过指定长度



使用起来也很简单：

```
1. > XADD mystream MAXLEN 2 * value 1 

2. 1526654998691-0 

3. > XADD mystream MAXLEN 2 * value 2 

4. 1526654999635-0 

5. > XADD mystream MAXLEN 2 * value 3 

6. 1526655000369-0 

7. > XLEN mystream 

8. (integer) 2 

9. > XRANGE mystream - + 

10. 1) 1) 1526654999635-0 

11.  2) 1) "value" 

12.    2) "2" 

13. 2) 1) 1526655000369-0 

14.  2) 1) "value" 

15.    2) "3" 
```



如果使用 MAXLEN 选项，当 Stream 的达到指定长度后，老的消息会自动被淘汰掉，因此 Stream 的大小是恒定的。目前还没有选项让 Stream 只保留给定数量的条目，因为为了一致地运行，这样的命令必须在很长一段时间内阻塞以淘汰消息。(例如在添加数据的高峰期间，你不得不长暂停来淘汰旧消息和添加新的消息)



另外使用 MAXLEN 选项的花销是很大的，Stream 为了节省内存空间，采用了一种特殊的结构表示，而这种结构的调整是需要额外的花销的。所以我们可以使用一种带有 ~ 的特殊命令：

```
 // 示例

1. XADD mystream MAXLEN ~ 1000 * ... entry fields here ... 
```



它会基于当前的结构合理地对节点执行裁剪，来保证至少会有 1000 条数据，可能是 1010 也可能是 1030。



##### 2.2 Stream怎么避免消息丢失？

在客户端消费者读取 Stream 消息时，Redis 服务器将消息回复给客户端的过程中，客户端突然断开了连接，消息就丢失了。但是 PEL 里已经保存了发出去的消息 ID，待客户端重新连上之后，可以再次收到 PEL 中的消息 ID 列表。不过此时 xreadgroup 的起始消息 ID 不能为参数 > ，而必须是任意有效的消息 ID，一般将参数设为 0-0，表示读取所有的 PEL 消息以及自 last_delivered_id 之后的新消息。



#### 3. Stream和kafka简析

Redis 基于内存存储，这意味着它会比基于磁盘的 Kafka 快上一些，也意味着使用 Redis 我们 **不能长时间存储大量数据**。不过如果您想以 **最小延迟** 实时处理消息的话，您可以考虑 Redis，但是如果 **消息很大并且应该重用数据** 的话，则应该首先考虑使用 Kafka。



另外从某些角度来说，**Redis Stream 也更适用于小型、廉价的应用程序**，因为 Kafka 相对来说更难配置一些