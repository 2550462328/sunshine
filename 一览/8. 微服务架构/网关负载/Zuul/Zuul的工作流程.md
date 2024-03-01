![Zuul 原理](http://static.iocoder.cn/20944a735d250d7d3338fe9deea179f8)



其中Filter的分类有：

1. PRE Filter : PRE Filter 在请求被路由之前调用 。 一般用于实现身份验证、资源 审查、记录调试信息等。
2. ROUTING Filter：ROUTING Filter 将请求路由 到微服务实例，该 Filter 用于构 建发送给微服务实例的请求 ， 并使用 Apache HTTPClient 或 Netflix Ribbon 请求微服务实 例 。
3. POST Filter : POST Filter 一般用来为响应添加标准的 HTTP Headr、收集统计 信息和指标 ， 以及将响应从微服务发送给到客户端等，该 Filter 在将请求路由到微服务实 例以后被执行 。
4. ERROR Filter：在其他 阶段发生错误时执行 ERROR Filter。