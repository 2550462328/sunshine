键空间通知功能自**2.8.0版本**开始可用



接收事件的类型有：

1. 所有影响给定键的命令。
2. 所有接收LPUSH操作（将一个或多个值插入到列表头部。 如果 key不存在，一个空列表会被创建并执行 LPUSH 操作。 当 key存在但不是列表类型时，返回一个错误）的键。
3. 所有在数据库中到期的键（expired）。

注意：这种事件通知并**不可靠**，也就是说，如果你的发布/订阅客户端断开连接，并在稍后重连，那么所有在客户端断开期间发送的事件将会丢失。因为redis的发布/订阅模式是**fire and forget**。

官方准备解决手段：

1. 为发布/订阅本身带来可靠性；
2. 允许Lua脚本拦截发布/订阅的消息以执行推送等操作，就像往队列里推送事件一样。



键事件类型有：

- 键空间通知：PUBLISH _*keyspace@0_*:mykey del，接收到的消息是事件的名称。
- 键事件通知：PUBLISH _*keyevent@0_*:del mykey，接收到的消息是键的名称。



默认情况下键事件是不启用了，需要在redis.windows.conf（或者redis.conf）中修改（没有就新增），关键参数**notify-keyspace-events**



参数后面（没有冒号）的值定义了开启监听事件类型（不用隔开）如下：

![img](http://pcc.huitogo.club/f7ffad2f322aa7978820cd4b29e554a3)

其中K事件和E事件**至少存在一个**，比如集合命令参数是Ks。字符串KEA可以所有可能的事件。

**所有命令仅在真正修改目标键时才生成事件**。例如，使用SREM命令从集合中删除一个不存在的元素将不会改变键的值，因此不会生成任何事件。



注：在过期事件中，过期事件触发的条件：

1. 当命令访问键时，发现键已过期。如果没有命令不断地访问键，并且有很多键都有关联的TTL，那么在键的生存时间降至零到生成expired事件之间，将会有明显的延迟。
2. 通过后台系统在后台逐步查找过期的键，以便能够收集那些从未被访问的键。



无法保证Redis服务器在键过期的那一刻同时生成expired事件。基本上，**expired事件是在Redis服务器删除键的时候生成的**，而不是在理论上生存时间达到零值时生成的。

跟处理失效券场景的联系：可以将优惠卷放进redis中，设置失效时间，在优惠券失效即redis触发过期事件时修改优惠券状态（或者其他业务）