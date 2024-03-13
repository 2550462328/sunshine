Spring Cloud组件最早是Netflix研发的

![Spring Cloud的 组件](https://pcc.huitogo.club/z0/4935fcc0a209fd1d4b70cade94986f59)



我们最为熟知的，可能就是 Spring Cloud Netflix ，它是 Netflix 公司基于它们自己的 Eureka、Hystrix、Zuul、Ribbon 等组件，构建的一个 Spring Cloud 实现技术栈。

![img](https://pcc.huitogo.club/z0/20181226104611145.png)



Eureka + Ribbon + Feign + Hystrix + Zuul 交互图：

![Eureka + Ribbon + Feign + Hystrix + Zuul](https://pcc.huitogo.club/z0/64e9a7827c76d38f899160da6f736ea2)

但是

Spring Cloud Netflix项目已经进入维护模式

> 最近，Netflix 宣布 Hystrix正在进入维护模式。自2016年以来，Ribbon已处于类似状态。虽然Hystrix和Ribbon现已处于维护模式，但它们仍然在Netflix大规模部署。
>
> Hystrix Dashboard和Turbine已被Atlas取代。这些项目的最后一次提交别是2年和4年前。Zuul 1和Archaius 1都被后来不兼容的版本所取代。
>
> 以下Spring Cloud Netflix模块和相应的Starter将进入维护模式：
>
> spring-cloud-netflix-archaius
> spring-cloud-netflix-hystrix-contract
> spring-cloud-netflix-hystrix-dashboard
> spring-cloud-netflix-hystrix-stream
> spring-cloud-netflix-hystrix
> spring-cloud-netflix-ribbon
> spring-cloud-netflix-turbine-stream
> spring-cloud-netflix-turbine
> spring-cloud-netflix-zuul
> 这不包括Eureka或并发限制模块。
>
> 什么是维护模式？
>
> 将模块置于维护模式，意味着Spring Cloud团队将不会再向模块添加新功能。我们将修复block级别的bug以及安全问题，我们也会考虑并审查社区的小型pull request。
>
> 我们打算继续支持这些模块，直到 Greenwich release 被普遍采用至少一年。
>
> 替代品
>
> 我们建议对这些模块提供的功能进行以下替换
>
> CURRENT	REPLACEMENT
> Hystrix	Resilience4j
> Hystrix Dashboard / Turbine	Micrometer + Monitoring System
> Ribbon	Spring Cloud Loadbalancer
> Zuul 1	Spring Cloud Gateway
> Archaius 1	Spring Boot external config + Spring Cloud Config
> 寻找关于Spring Cloud Loadbalancer的未来博客文章，并与新的Netflix项目Concurrency Limits集成。



所幸我们可以在Spring Cloud Alibaba和其他第三方应用中找到替代品

|          | Netflix | Alibaba     | 其它                      |
| :------- | :------ | :---------- | ------------------------- |
| 注册中心 | Eureka  | Nacos       | Zookeeper、Consul、Etcd   |
| 熔断器   | Hystrix | Sentinel    | Resilience4j              |
| 网关     | Zuul1   | 暂无        | Spring Cloud Gateway      |
| 负载均衡 | Ribbon  | Dubbo(未来) | Spring-cloud-loadbalancer |

其它组件，例如配置中心、链路追踪、服务引用等等，都有相应其它的实现。