在 Spring Cloud 中，目前使用的声明式调用组件，只有：

- `spring-cloud-openfeign` ，基于 Feign 实现。

如果熟悉 Dubbo 的会知道，Dubbo 的 Service API 接口，也是一种声明式调用的提现。

> 注意，Feign 并非 Netflix 团队开发的组件。所有基于 Netflix 组件都在 `spring-cloud-netflix` 项目下噢。



Feign 是受到 Retrofit、JAXRS-2.0 和 WebSocket 启发的 Java 客户端联编程序。Feign 的主要目标是将Java Http 客户端变得简单。



**Feign的工作原理：**

![Feign 原理](http://static.iocoder.cn/6650aa32de0def76db0e4c5228619aef)

- **Feign的一个关键机制就是使用了动态代理**
- 首先，如果你对某个接口定义了 `@FeignClient` 注解，Feign 就会针对这个接口创建一个动态代理。
- 接着你要是调用那个接口，本质就是会调用 Feign 创建的动态代理，这是核心中的核心。
- Feig n的动态代理会根据你在接口上的 `@RequestMapping` 等注解，来动态构造出你要请求的服务的地址。
- 最后针对这个地址，发起请求、解析响应。



