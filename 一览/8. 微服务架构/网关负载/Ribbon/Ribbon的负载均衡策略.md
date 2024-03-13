Ribbon作为后端负载均衡器，比Nginx更注重的是承担并发而不是请求分发，可以直接感知后台动态变化来指定分发策略。



它一共提供了7种负载均衡策略，其中Round Robin 算法是默认策略：

- **BestAvailableRule**

> 选择一个最小的并发请求的server

实现说明：逐个考察Server，如果Server被tripped了，则忽略，在选择其中`ActiveRequestsCount`最小的server



- **AvailabilityFilteringRule**

> 过滤掉那些因为一直连接失败的被标记为circuit tripped的后端server，并过滤掉那些高并发的的后端server（active connections 超过配置的阈值）

实现说明：使用一个`AvailabilityPredicate`来包含过滤server的逻辑，其实就就是检查status里记录的各个server的运行状态



- **WeightedResponseTimeRule**

> 根据响应时间分配一个weight，响应时间越长，weight越小，被选中的可能性越低。

实现说明：一个后台线程定期的从status里面读取评价响应时间，为每个server计算一个weight。Weight的计算也比较简单responsetime 减去每个server自己平均的responsetime是server的权重。当刚开始运行，没有形成status时，使用roubine策略选择server。



- **RetryRule**

> 对选定的负载均衡策略机上重试机制。

实现说明：在一个配置时间段内当选择server不成功，则一直尝试使用subRule的方式选择一个可用的server



- **RoundRobinRule**

> roundRobin方式轮询选择server

实现说明：轮询index，选择index对应位置的server



- **RandomRule**

> 随机选择一个server

实现说明：在index上随机，选择index对应位置的server



- **ZoneAvoidanceRule**

> 复合判断server所在区域的性能和server的可用性选择server

实现说明：使用`ZoneAvoidancePredicate`和`AvailabilityPredicate`来判断是否选择某个server，前一个判断判定一个zone的运行性能是否可用，剔除不可用的zone（的所有server），`AvailabilityPredicate`用于过滤掉连接数过多的Server。



源码主要是`LoadBalancerInterceptor`的拦截逻辑：

这里我们希望调用的服务地址是`http://user-service/user/`

![在这里插入图片描述](https://pcc.huitogo.club/z0/20190407154202955.png)



execute方法，主要是`#getLoadBalancer(serviceId)`

![在这里插入图片描述](https://pcc.huitogo.club/z0/20190407154302110.png)

- 这里已经将我们访问的地址变成了127.0.0.1:8081