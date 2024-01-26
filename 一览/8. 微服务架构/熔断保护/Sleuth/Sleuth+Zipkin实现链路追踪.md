#### 1. 为什么要追踪服务？

微服务架构上通过业务来划分服务的，通过REST调用，对外暴露的一个接口，可能需要很多个服务协同才能完成这个接口功能，如果链路上任何一个服务出现问题或者网络超时，都会形成导致接口调用失败。随着业务的不断扩张，服务之间互相调用会越来越复杂，所以需要追踪服务调用关系，以便可视化服务关系进行错误排查。



#### 2. 服务链路追踪中的专业术语

**1）Span**：基本工作单元，例如，在一个新建的span中发送一个RPC等同于发送一个回应请求给RPC，span通过一个64位ID唯一标识，span还有其他数据信息，比如摘要、时间戳事件、关键值注释(tags)、span的ID、以及进度ID(通常是IP地址)

**2）Trace**：一系列spans组成的一个树状结构，例如，如果你正在跑一个分布式大数据工程，你可能需要创建一个trace。

**3）Annotation**：用来及时记录一个事件的存在，一些核心annotations用来定义一个请求的开始和结束

- cs - Client Sent -客户端发起一个请求，这个annotion描述了这个span的开始
- sr - Server Received 服务端获得请求并准备开始处理它，如果将其sr减去cs时间戳便可得到网络延迟
- ss - Server Sent 注解表明请求处理的完成(当请求返回客户端)，如果ss减去sr时间戳便可得到服务端需要的处理请求时间
- cr - Client Received 表明span的结束，客户端成功接收到服务端的回复，如果cr减去cs时间戳便可得到客户端从服务端获取回复的所有所需时间



#### 3. server-zipkin介绍

使用server-zipkin来收集服务调用数据，前提是服务之间发生了调用事件，并且构建简单的可视化服务调用页面。

将Span和Trace在一个系统中使用Zipkin注解的过程图形化：

![img](http://pcc.huitogo.club/0512c41088c73a3c099d104df028bf9e)



#### 4. 如何使用server-zipkin做服务链路追踪

1）下载server-zipkin.exe.jar，并且启动

2）在需要追踪的服务中添加spring-cloud-starter-zipkin依赖，并且配置连接的server-zipkin地址

```
spring.zipkin.base-url=http://localhost:9411
```



3）进行服务调用后打开http://localhost:9411/ 的界面就可以看到服务调用关系了

![img](http://pcc.huitogo.club/b24320f2f1262d789b230b431b8ccb7c)



下面是一个trace的过程

![img](http://pcc.huitogo.club/d05e7ac51bdf9d7b6784b09bf6e91bea)