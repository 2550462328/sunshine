访问Eureka主页时，如果看到这样一段大红色的句子：

> `EMERGENCY! EUREKA MAY BE INCORRECTLY CLAIMING INSTANCES ARE UP WHEN THEY’RE NOT. RENEWALS ARE LESSER THAN THRESHOLD AND HENCE THE INSTANCES ARE NOT BEING EXPIRED JUST TO BE SAFE.`

那么就表明着Eureka的 自我保护模式(self-preservation mode) 被启动了，当 Eureka Server 节点在短时间内丢失了过多实例的连接时（比如网络故障或频繁的启动关闭客户端），那么这个节点就会进入自我保护模式，一旦进入到该模式，Eureka server 就会保护服务注册表中的信息，不再删除服务注册表中的数据（即不会注销任何微服务），当网络故障恢复后，该 Ereaka Server 节点就会自动退出自我保护模式（我的 Eureka Server 已经几个月了，至今未自动退出该模式）



默认情况下，如果 Ereaka Server 在一段时间内没有接受到某个微服务示例的心跳，便会注销该实例（默认90秒），而一旦进入自我保护模式，那么即使你关闭了指定实例，仍然会发现该 Ereaka Server 的注册实例中会存在被关闭的实例信息，如果你对该实例做了负载均衡，那么仅关闭了其中一个实例，则通过网关调用接口api时很可能会发生如下异常：

```
{
    "timestamp": 1507707671780,
    "status": 500,
    "error": "Internal Server Error",
    "exception": "com.netflix.zuul.exception.ZuulException",
    "message": "GENERAL"
}
```



解决这种情况的方法主要有几种方式：

1. 等待 Eureka Server 自动恢复
   
正常的情况下，等待网络恢复（或者没有频繁的启动与关闭实例）后，等待一段时间 Eureka Server 会自动关闭自我保护模式，但是如果它迟迟没有关闭该模式，那么便可以尝试手动关闭，如下。
   
2. 重启 Eureka Server

   通常而言，PRD 环境建议对 Eureka Server 做负载均衡，这样在依次关闭并开启 Eureka Server 后，无效的实例会被清除，并且不会对正常的使用照成影响。

3. 关闭 Eureka 的自我保护模式

   在yml配置文件中新增如下配置：

```
eureka:
  server:
    enable-self-preservation: false
    eviction-interval-timer-in-ms: 4000
```

从根源解决问题，但是并不推荐在PRD环境中使用，关闭自我保护模式，会有另外一个可能的问题，即隔一段时间后，可能会发生实例并未关闭，却无法通过网关访问了，此时很可能是由于网络问题，导致实例（或网关）与 Eureka Server 断开了连接，Eureka Server 已经将其注销（网络恢复后，实例并不会再次注册），此时重启 Eureka Server 节点或实例，并等待一小段时间即可。

