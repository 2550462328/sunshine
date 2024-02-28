#### 1. 功能组件

![图1](https://img-blog.csdnimg.cn/20201011011947309.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MTAyOTI4Ng==,size_16,color_FFFFFF,t_70#pic_center)

其中各组件功能如下：

- Provider: 暴露服务的服务提供方

- Protocol 负责提供者和消费者之间协议交互数据

- Service 真实的业务服务信息 可以理解成接口 和 实现

- Container Dubbo的运行环境

- Consumer: 调用远程服务的服务消费方

- Protocol 负责提供者和消费者之间协议交互数据

- Cluster 感知提供者端的列表信息

- Proxy 可以理解成 提供者的服务调用代理类 由它接管 Consumer中的接口调用逻辑

- Registry: 注册中心，用于作为服务发现和路由配置等工作，提供者和消费者都会在这里进行注册

- Monitor: 用于提供者和消费者中的数据统计，比如调用频次，成功失败次数等信息。



#### 2. 组件交互

![详细调用图](http://static.iocoder.cn/images/Dubbo/2017_10_24/01.png)

- 注意，图中的【代理】指的是 **proxy 代理服务层**，和 Consumer 或 Provider 在同一进程中。

- 注意，图中的【负载均衡】指的是 **cluster 路由层**，和 Consumer 或 Provider 在同一进程中。

  

组件之间交互的大概流程如下：

1. Provider

   - 第 0 步，start 启动服务。
   - 第 1 步，register 注册服务到注册中心。
2. Consumer

   - 第 2 步，subscribe 向注册中心订阅服务。
     - 注意，只订阅使用到的服务。
     - 再注意，首次会拉取订阅的服务列表，缓存在本地。
   - 【异步】第 3 步，notify 当服务发生变化时，获取最新的服务列表，更新本地缓存。
3. invoke 调用

   - Consumer 直接发起对 Provider 的调用，无需经过注册中心。而对多个 Provider 的负载均衡，Consumer 通过 **cluster** 组件实现。
4. count 监控

   - 【异步】Consumer 和 Provider 都异步通知监控中心。



大概的总结流程图：

![image-20240227113052499](C:\Users\huizhang43\AppData\Roaming\Typora\typora-user-images\image-20240227113052499.png)



#### 3. 服务链路调用

<img src="https://img-blog.csdnimg.cn/20201011151807739.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MTAyOTI4Ng==,size_16,color_FFFFFF,t_70#pic_center" alt="在这里插入图片描述" style="zoom: 50%;" />

> 其中**淡绿色**代表了 服务生产者的范围；**淡蓝色** 代表了服务消费者的范围；**红色箭头**代表了调用的方向
>



服务的调用链路如下：

1. 消费者通过Interface进行方法调用 统一交由消费者端的 Proxy 通过ProxyFactory 来进行代理 对象的创建 使用到了 jdk javassist技术

2. 交给Filter 这个模块 做一个统一的过滤请求

3. 接下来会进入最主要的Invoker调用逻辑

4. 通过Directory 去配置中新读取信息 最终通过list方法获取所有的Invoker

5. 通过Cluster模块 根据选择的具体路由规则 来选取Invoker列表

6. 通过LoadBalance模块 根据负载均衡策略 选择一个具体的Invoker 来处理我们的请求

7. 如果执行中出现错误 并且Consumer阶段配置了重试机制 则会重新尝试执行

8. 继续经过Filter 进行执行功能的前后封装 Invoker 选择具体的执行协议

9. 客户端 进行编码和序列化 然后发送数据

10. 到达Provider中的 Server 在这里进行 反编码 和 反序列化的接收数据

11. 使用Exporter选择执行器

12. 交给Filter 进行一个提供者端的过滤 到达 Invoker 执行器

13. 通过Invoker 调用接口的具体实现 然后返回




