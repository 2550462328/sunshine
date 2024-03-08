这里只提供Eureka生产环境配置的一些思路

#### 1. 服务获取中的缓存问题

**为什么修改client的默认心跳时间，会导致自我保护模式失效？**

Eureka Service会认为客户端是以30s的频率来发送心跳的。服务端期望收到的最大心跳时间是：
n instances x 2(60s/30s) x threshold

如果是2个实例，Eureka会期望每分钟有：2 instances x 2 x 85% =3.4个心跳,也就是说需要3个心跳。
如果client的心跳改成15s，挂掉一个，另一个在1min内会发出4个心跳，而这时候的阈值还是3.4个，自我保护模式就失效了。
核心原因就是在Eureka Server计算期望心跳数的时候写死了每分钟的心跳间隔，即30秒，所以他永远会是*2(



这里有一个参数可以调整，eureka.server.renewalThresholdUpdateIntervalMs，心跳阈值重新计算的周期，默认15分钟，可以改短一点，2min



**客户端首次注册时间为什么要30s？如何改进？**

首次注册行为是和首次心跳绑定在一起的，首次心跳发送以后会收到not found的响应,client就知道还没注册过，client就会马上注册。首次心跳由参数eureka.instance.leaseRenewalIntervalInSeconds控制的，默认30

可以通过eureka.client.initialInstanceInfoReplicationIntervalSeconds参数来加快首次注册的速度。他是控制首次改变实例状态（UP/DOWN ）的时间，启动的时候状态肯定是需要改变的，所以他可以用来加快首次注册速度，并且改变这个值不会影响到保护模式

另外如果你使用的是spring cloud eureka的话没首次注册延迟的问题，他会马上注册



**其他影响快速获取服务信息的因素**

- **【服务端缓存】**
  因为服务端默认会有个read only response cache（下面会细说），每30秒更新一次(eureka.server.response-cache-update-interval-ms),所以可能注册了不是马上能看到（虽然通过rest api不能看到，但是你可以在web ui上看到，因为ui没有缓存）

- **【客户端缓存】**
  Eureka Client缓存的定期更新周期，他由eureka.client.registryFetchIntervalSeconds控制，默认30秒， 改成5秒

- **【Ribbon缓存】**
  如果你采用Ribbon来访问服务，那么这里会有个缓存（他的数据来源是本地Eureka Client缓存），他由ribbon. ServerListRefreshInterval控制，默认30秒， 改成2秒



**怎么更快的踢掉没有心跳的机器**

eureka.instance.leaseExpirationDurationInSeconds，这个值用来控制多久踢掉机器，默认是3个心跳周期，有点久，可以考虑改成2个，他不会影响到保护模式（如果开启自我保护模式，心跳间隔因为上面的bug不能改，只能改这个了 -_-）



#### 2. 服务端缓存细节

Eureka内部的缓存分很多级，主要有registry、readWriterCacheMap、readOnlyCacheMap；另外还有一个维护最近180s增量的队列recentlyChangedQueue

- **写操作**

包括注册、取消注册等，都直接操作在registry上，同时也会更新recentlyChangedQueue和readWriterCacheMap

- **读操作**

读默认是从readOnlyCacheMap读取，读不到的话再从readWriterCacheMap，还没有再从registry

- **滥用缓存的读操作**

这个读操作的三级缓存结构，非常让人困惑，registry已经是ConcurrentHashMap，纯内存操作，性能非常高了，为什么前面还要加两级缓存；readWriterCacheMap的数据是在写入以后responseCacheAutoExpirationInSeconds(默认180)秒内失效，readOnlyCacheMap则是一个定时任务，每responseCacheUpdateIntervalMs(默认30)秒从readWriterCacheMap获取最新数据



**去掉readOnlyCacheMap**

从CAP理论上看，Eureka是一个AP系统，但是在C层面这么弱，就是因为各种无谓的缓存造成的，看了下readWriterCacheMap去掉比较难，但是readOnlyCacheMap有一个开关useReadOnlyResponseCache，果断关掉！！



#### 3. Time Lag

最后再来看下Eureka wiki中提到的2min time lag问题，其实分多个角度看，不一定是2min

- **服务正常上线/修改，最大可能会有120s滞后**

```
- 30(首次注册 init registe) + 30(readOnlyCacheMap)+30(client fetch interval)+30(ribbon)=120
- 如果是在Spring Cloud环境下使用这些组件(Eureka, Ribbon)，不会有首次注册30秒延迟的问题，服务启动后会马上注册,所以从注册到发现，最多可能是90s。
```



- **服务异常下线：最大可能会有270s滞后**

```
- 定时清理任务每eureka.server. evictionIntervalTimerInMs(默认60)执行一次清理任务
- 每次清理任务会把90秒(3个心跳周期，eureka.instance.leaseExpirationDurationInSeconds)没收到心跳的踢除，但是根据官方的说法 ，因为代码实现的bug，这个时间其实是两倍，即180秒，也就是说如果一个客户端因为网络问题或者主机问题异常下线，可能会在180秒后才剔除
- 读取端，因为readOnlyCacheMap以及客户端缓存的存在，可能会在30(readOnlyCacheMap)+30(client fetch interval)+30(ribbon)=90
- 所以极端情况最终可能会是180+90=270
```



#### 4. 生产环境配置示例

推荐的生产环境最佳配置如下：（可用于中小规模环境）：



- **服务提供者和client配置**

```
## 心跳间隔，5秒
eureka.instance.leaseRenewalIntervalInSeconds=5
## 没有心跳的淘汰时间，10秒
eureka.instance.leaseExpirationDurationInSeconds=10

# 定时刷新本地缓存时间
eureka.client.registryFetchIntervalSeconds=5
# ribbon缓存时间
ribbon.ServerListRefreshInterval=2000
```



- **Eureka Server端配置**

```
## 中小规模下，自我保护模式坑比好处多，所以关闭它
eureka.server.enableSelfPreservation=false
## 心跳阈值计算周期，如果开启自我保护模式，可以改一下这个配置
## eureka.server.renewalThresholdUpdateIntervalMs=120000

## 主动失效检测间隔,配置成5秒
eureka.server.evictionIntervalTimerInMs=5000

## 心跳间隔，5秒
eureka.instance.leaseRenewalIntervalInSeconds=5
## 没有心跳的淘汰时间，10秒
eureka.instance.leaseExpirationDurationInSeconds=10

## 禁用readOnlyCacheMap
eureka.server. useReadOnlyResponseCache=false
```



- 改成上面配置后:
  - 正常上线下线客户端最大感知时间：eureka.client.registryFetchIntervalSeconds+ribbon. ServerListRefreshInterval = 7秒
  - 异常下线客户端最大感知时间：
    2*eureka.instance.leaseExpirationDurationInSeconds+
    eureka.server.evictionIntervalTimerInMs+
    eureka.client.registryFetchIntervalSeconds+
    ribbon. ServerListRefreshInterval = 32