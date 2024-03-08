#### 1. 消费端集群容错策略

即消费者请求生产者服务失败后应该怎么做

- **Failover Cluster 模式**

失败自动切换，自动重试其他机器，**默认**就是这个，常见于读操作。（失败重试其它机器）

可以通过以下几种方式配置重试次数：

```xml
<dubbo:service retries="2" />
```

或者

```xml
<dubbo:reference retries="2" />
```

或者

```xml
<dubbo:reference>
    <dubbo:method name="findFoo" retries="2" />
</dubbo:reference>
```



- **Failfast Cluster 模式**

一次调用失败就立即失败，常见于非幂等性的写操作，比如新增一条记录（调用失败就立即失败）



- **Failsafe Cluster 模式**

出现异常时忽略掉，常用于不重要的接口调用，比如记录日志。

配置示例如下：

```xml
<dubbo:service cluster="failsafe" />
```

或者

```xml
<dubbo:reference cluster="failsafe" />
```



- **Failback Cluster 模式**

失败了后台自动记录请求，然后定时重发，比较适合于写消息队列这种。



- **Forking Cluster 模式**

**并行调用**多个 provider，只要一个成功就立即返回。常用于实时性要求比较高的读操作，但是会浪费更多的服务资源，可通过 `forks="2"` 来设置最大并行数。



- **Broadcast Cluster 模式**

逐个调用所有的 provider。任何一个 provider 出错则报错（从 `2.1.0` 版本开始支持）。通常用于通知所有提供者更新缓存或日志等本地资源信息。



总结：实际场景下，我们一般会**禁用掉重试**。因为，因为超时后重试会有问题，超时你不知道是成功还是失败。例如，可能会导致两次扣款的问题。

所以，我们一般使用 failfast 集群容错策略，而不是 failover 策略。

当然，可以将操作分成**读**和**写**两种，前者支持重试，后者不支持重试。因为，**读**操作天然具有幂等性。



#### 2. 注册中心挂了还可以通信吗？

- consumer

  从consumer来说，在首次启动时会拉取provider列表缓存在本地，即使注册中心挂了也能从缓存中找到provider地址进行调用；

  另外consumer也会对proverder列表持久化，即使重启也能从本地拉取到缓存中。

- provider

  从provider来说，如果 Provider 是**正常关闭**，它会主动且直接对和其处于连接中的 Consumer 们，发送一条“我要关闭”了的消息。那么，Consumer 们就不会调用该 Provider ，而调用其它的 Provider ；

  如果Provider是非正常关闭，consumer在调用provider失败后会通过**集群容错策略**切换到另一台provider；