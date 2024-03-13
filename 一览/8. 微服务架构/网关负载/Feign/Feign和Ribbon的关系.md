**Feign和Ribbon的关系：**

- Feign           Feign is a factory for generating targeted http apis
- FeignFacrtoryBean     代理FeignService
- Client     HttpClient   怎么发送请求
- Target     目前地址
- LoadBalancer     ribbon负载均衡

![img](https://pcc.huitogo.club/z0/252461fbb6d64d3dbc1914b7eadbfb86)

- 首先，用户调用 Feign 创建的动态代理。
- 然后，Feign 调用 Ribbon 发起调用流程。

​        首先，Ribbon 会从 Eureka Client 里获取到对应的服务列表。

​        然后，Ribbon 使用负载均衡算法获得使用的服务。

这可能是比较绕的，因为 Feign 和 Ribbon 都存在使用 HTTP 库调用指定的服务，那么两者在集成之后，必然是只能保留一个。比较正常的理解，也是保留 Feign 的调用，而 Ribbon 更纯粹的只负责负载均衡的功能。