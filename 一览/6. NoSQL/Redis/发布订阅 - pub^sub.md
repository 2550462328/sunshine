#### 1. 什么是发布订阅？

发布/ 订阅系统 是 Web 系统中比较常用的一个功能。简单点说就是 发布者发布消息，订阅者接受消息，这有点类似于我们的报纸/ 杂志社之类的：

![img](http://pcc.huitogo.club/6983190b737fab0d70f5727c4085034a)



在redis中我们虽然可以使用一个 list 列表结构结合 lpush 和 rpop 来实现消息队列的功能，但是似乎很难实现实现 **消息多播** 的功能：

![img](http://pcc.huitogo.club/f66909308432c5930dce044460637bf1)



为了支持消息多播，Redis 不能再依赖于那 5 种基础的数据结构了，它单独使用了一个模块来支持消息多播，这个模块就是 PubSub，也就是 PublisherSubscriber (发布者/ 订阅者模式)。



我们从 上面的图 中可以看到，基于 list 结构的消息队列，是一种 Publisher 与 Consumer 点对点的强关联关系，Redis 为了消除这样的强关联，引入了另一种概念：**频道** (channel)：

![img](http://pcc.huitogo.club/c29a6f469e779aa5cbafff4bb51b9960)



当 Publisher 往 channel 中发布消息时，关注了指定 channel 的 Consumer 就能够同时受到消息。但这里的 **问题 是，消费者订阅一个频道是必须 明确指定频道名称** 的，这意味着，如果我们想要 **订阅多个 频道**，那么就必须 显式地关注多个 名称。



为了简化订阅的繁琐操作，Redis 提供了 **模式订阅** 的功能 Pattern Subscribe，这样就可以 一次性关注多个频道 了，即使生产者新增了同模式的频道，消费者也可以立即受到消息：

![img](http://pcc.huitogo.club/6123d50894eae2ef99fb451d91ed6e2b)



在上图中wmyskxz.*的模式订阅 可以同时接受到wmyskxz.chat和wmyskxz.log频道发布的消息。



#### 2. 具体怎么使用？

在Redis的PubSub模块的使用常用命令如下：

```
1. # 订阅频道： 

2. SUBSCRIBE channel [channel ....]  # 订阅给定的一个或多个频道的信息 

3. PSUBSCRIBE pattern [pattern ....] # 订阅一个或多个符合给定模式的频道 

4. # 发布频道： 

5. PUBLISH channel message # 将消息发送到指定的频道 

6. # 退订频道： 

7. UNSUBSCRIBE [channel [channel ....]]  # 退订指定的频道 

8. PUNSUBSCRIBE [pattern [pattern ....]] #退订所有给定模式的频道 
```



#### 3. Redis的发布订阅实现原理是什么？

**每个 Redis 服务器进程维持着一个标识服务器状态 的 redis.h/redisServer 结构，其中就 保存着有订阅的频道** 以及 订阅模式 的信息：

```
1. struct redisServer { 

2.   // ... 

3.   dict *pubsub_channels; // 订阅频道 

4.   list *pubsub_patterns; // 订阅模式 

5.   // ... 

6. }; 
```



##### 3.1 订阅频道

当客户端订阅某一个频道之后，Redis 就会往 pubsub_channels 这个字典中新添加一条数据，实际上这个 dict 字典维护的是一张链表，比如，下图展示的 pubsub_channels 示例中，client 1、client 2 就订阅了 channel 1，而其他频道也分别被其他客户端订阅：

![img](http://pcc.huitogo.club/ab310edf1e2439c4185343bbbd493b08)



- **SUBSCRIBE 命令**

SUBSCRIBE 命令的行为可以用下列的伪代码表示：

```
1. def SUBSCRIBE(client, channels): 

2.   # 遍历所有输入频道 

3.   for channel in channels: 

4.     # 将客户端添加到链表的末尾 

5.     redisServer.pubsub_channels[channel].append(client) 
```



通过 pubsub_channels 字典，程序只要检查某个频道是否为字典的键，就可以知道该频道是否正在被客户端订阅；只要取出某个键的值，就可以得到所有订阅该频道的客户端的信息。



- **PUBLISH 命令**

了解 SUBSCRIBE，那么 PUBLISH 命令的实现也变得十分简单了，只需要通过上述字典定位到具体的客户端，再把消息发送给它们就好了：(伪代码实现如下)

```
1. def PUBLISH(channel, message): 

2.   # 遍历所有订阅频道 channel 的客户端 

3.   for client in server.pubsub_channels[channel]: 

4.     # 将信息发送给它们 

5.     send_message(client, message) 
```



- **UNSUBSCRIBE 命令**

使用 UNSUBSCRIBE 命令可以退订指定的频道，这个命令执行的是订阅的反操作：它从 pubsub_channels 字典的给定频道（键）中，删除关于当前客户端的信息，这样被退订频道的信息就不会再发送给这个客户端。



##### 3.2 订阅模式

![img](http://pcc.huitogo.club/1e8e18634eb4fac1e26cc555a2c36c3b)



正如我们上面说到了，当发送一条消息到 wmyskxz.chat 这个频道时，Redis 不仅仅会发送到当前的频道，还会发送到匹配于当前模式的所有频道，实际上，pubsub_patterns 背后还维护了一个 redis.h/pubsubPattern 结构：

```
1. typedef struct pubsubPattern { 

2.   redisClient *client; // 订阅模式的客户端 

3.   robj *pattern;    // 订阅的模式 

4. } pubsubPattern; 
```



每当调用 PSUBSCRIBE 命令订阅一个模式时，程序就创建一个包含客户端信息和被订阅模式的 pubsubPattern 结构，并将该结构添加到 redisServer.pubsub_patterns 链表中。



我们来看一个 pusub_patterns 链表的示例：

![img](http://pcc.huitogo.club/f44d0a79a07450657a61216c4738e20d)



这个时候客户端 client 3 执行 PSUBSCRIBE wmyskxz.java.*，那么 pubsub_patterns 链表就会被更新成这样：

![img](http://pcc.huitogo.club/77fd9b9763170150b273befb2bbbd4d7)



通过遍历整个 pubsub_patterns 链表，程序可以检查所有正在被订阅的模式，以及订阅这些模式的客户端。



- **PUBLISH 命令**

上面给出的伪代码并没有 完整描述 PUBLISH 命令的行为，因为 PUBLISH 除了将 message 发送到 所有订阅 channel 的客户端 之外，它还会将 channel 和 pubsub_patterns 中的 模式 进行对比，如果 channel 和某个模式匹配的话，那么也将 message 发送到 订阅那个模式的客户端。



完整描述 PUBLISH 功能的伪代码定于如下：

```
1. def PUBLISH(channel, message): 

2.   # 遍历所有订阅频道 channel 的客户端 

3.   for client in server.pubsub_channels[channel]: 

4.     # 将信息发送给它们 

5.     send_message(client, message) 

6.   # 取出所有模式，以及订阅模式的客户端 

7.   for pattern, client in server.pubsub_patterns: 

8.     # 如果 channel 和模式匹配 

9.     if match(channel, pattern): 

10.       # 那么也将信息发给订阅这个模式的客户端 

11.       send_message(client, message) 
```



- **PUNSUBSCRIBE 命令**

使用 PUNSUBSCRIBE 命令可以退订指定的模式，这个命令执行的是订阅模式的反操作：序会删除 redisServer.pubsub_patterns 链表中，所有和被退订模式相关联的 pubsubPattern 结构，这样客户端就不会再收到和模式相匹配的频道发来的信息。



#### 4. Pub/Sub有什么不足之处吗？

尽管 Redis 实现了 PubSub 模式来达到了 多播消息队列 的目的，但在实际的消息队列的领域，几乎 找不到特别合适的场景，因为它的缺点十分明显：

1. **没有 Ack 机制**，也不保证数据的连续： PubSub 的生产者传递过来一个消息，Redis 会直接找到相应的消费者传递过去。如果没有一个消费者，那么消息会被直接丢弃。如果开始有三个消费者，其中一个突然挂掉了，过了一会儿等它再重连时，那么重连期间的消息对于这个消费者来说就彻底丢失了。
2. **不持久化消息**： 如果 Redis 停机重启，PubSub 的消息是不会持久化的，毕竟 Redis 宕机就相当于一个消费者都没有，所有的消息都会被直接丢弃。



基于上述缺点，Redis 的作者甚至单独开启了一个 **Disque** 的项目来专门用来做多播消息队列，不过该项目目前好像都没有成熟。不过后来在 2018 年 6 月，Redis 5.0 新增了 **Stream 数据结构**，这个功能给 Redis 带来了 持久化消息队列，**从此 PubSub 作为消息队列的功能可以说是就消失了..**