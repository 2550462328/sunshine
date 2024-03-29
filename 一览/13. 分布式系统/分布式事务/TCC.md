阿里巴巴提出了新的TCC协议，TCC协议将一个任务拆分成Try、Confirm、Cancel，正常的流程会先执行Try，如果执行没有问题，再执行Confirm，如果执行过程中出了问题，则执行操作的逆操Cancel

> TCC 模式本质也是 2PC ，只是 TCC 在应用层控制。
>



TCC协议系统交互过程如下

![img](http://pcc.huitogo.club/8caa259fe73187908460d3b26f7b72d3)

- Try:
  - 尝试执行业务
  - 完成所有业务检查（一致性）
  - 预留必须业务资源（准隔离性）
- Confirm:
  - 确认执行业务；
  - 真正执行业务，不作任何业务检查
  - 只使用Try阶段预留的业务资源
  - Confirm 操作满足幂等性

- Cancel:
  - 取消执行业务
  - 释放Try阶段预留的业务资源
  - Cancel操作满足幂等性

这三个阶段，都会按本地事务的方式执行。不同于 XA的prepare ，**TCC 无需将 XA 的投票期间的所有资源挂起**，因此极大的提高了吞吐量。



**TCC特点如下：**

- 并发度较高，无长期资源锁定。
- 开发量较大，需要提供Try/Confirm/Cancel接口。
- 一致性较好，不会发生SAGA已扣款最后又转账失败的情况
- TCC适用于订单类业务，对中间状态有约束的业务
  


从TCC的逻辑上看，可以说TCC是简化版的三阶段提交协议，解决了两阶段提交协议的阻塞问题，但是没有解决极端情况下会出现不一致和脑裂的问题，例如，如果在Cancel的时候一些参与者收到指令，而一些参与者没有收到指令，整个系统仍然是不一致的。

然而，TCC通过自动化补偿手段，会把需要人工处理的不一致情况降到到最少，也是一种非常有用的解决方案，阿里在内部的一些中间件上实现了TCC模式。



**TCC应用场景？**

下面对TCC模式下，A账户往B账户汇款100元为例子，对业务的改造进行详细的分析：

![img](https://pcc.huitogo.club/z0/4da5bc0df774ef90e97c6358eb7e632f)

- 汇款服务和收款服务分别需要实现，Try-Confirm-Cancel 接口，并在业务初始化阶段将其注入到 TCC 事务管理器中。

汇款服务

- Try：
  - 检查A账户有效性，即查看A账户的状态是否为“转帐中”或者“冻结”
  - 检查A账户余额是否充足
  - 从A账户中扣减 100 元，并将状态置为“转账中”
  - 预留扣减资源，将从 A 往 B 账户转账 100 元这个事件存入消息或者日志中
- Confirm：
  - 不做任何操作
- Cancel：
  - A 账户增加 100 元
  - 从日志或者消息中，释放扣减资源

收款服务

- Try：
  - 检查 B 账户账户是否有效；
- Confirm：
  - 读取日志或者消息，B 账户增加 100 元
  - 从日志或者消息中，释放扣减资源；
- Cancel：
  - 不做任何操作



由此可以看出，TCC 模型对业务的侵入强，改造的难度大。

但是，在需要前置资源锁定的场景，不得不使用 XA 或 TCC 的方式。再例如说，下单场景，在订单创建之前，需要扣除如下几个资源：

- 优惠劵
- 钱包余额
- 积分
- ….

那么，不得不进行前置多资源锁定，无非是使用 XA 的强锁，还是 TCC 的弱锁。



**解决方案？**

- TCC-Transaction ，听说喜马拉雅在用。具体的源码解析，可以看看 《TCC-Transaction 源码分析》 。
- Hmily ，具体的源码解析，可以看看 《Hmily 实现原理与源码解析系列 —— 精品合集》 。
- ByteTCC



**为什么不用TCC？**

这种方案说实话几乎很少人使用，我们用的也比较少，但是也有使用的场景。因为这个**事务回滚**实际上是**严重依赖于你自己写代码来回滚和补偿**了，会造成补偿代码巨大，非常之恶心。

比如说我们，一般来说跟**钱**相关的，跟钱打交道的，**支付**、**交易**相关的场景，我们会用 TCC，严格保证分布式事务要么全部成功，要么全部自动回滚，严格保证资金的正确性，保证在资金上不会出现问题。

而且最好是你的各个业务执行的时间都比较短。

但是说实话，一般尽量别这么搞，自己手写回滚逻辑，或者是补偿逻辑，实在太恶心了，那个业务代码是很难维护的。

![distributed-transacion-TCC](https://pcc.huitogo.club/z0/distributed-transaction-TCC.png)