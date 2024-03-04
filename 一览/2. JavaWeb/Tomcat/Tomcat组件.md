tomcat 的组件 和 组件之间的交互主要如下

![img](http://pcc.huitogo.club/5fc5d76298edbf10a46aa8369fac7df5)



**server**：整个servlet容器，一个tomcat对应一个server，一个server包含多个service，Server就掌管着多个Service的死活

server在tomcat中的实现类是：StandardServer

**service**：service是对外提供服务的， 一个service包含多个connector（接受请求的协议），和一个container（容器），多个connector共享一个container容器，

service在tomcat中的实现类是：StandardService

**connector**：connector主要用来接收请求，解析请求内容，封装request和response，然后将准备好的数据交给Container处理

**executor**：线程池

**container**：Container就是我们常说的容器，里面可以有多个Host,一个host表示一个虚拟主机，就是一个对应一个WebApps. 最后Container处理完请求之后会将响应内容返回给Connecter,再由Connecter返回给客户端包含engine，host，context，wrapper等组件

**engine**：Container（容器/引擎）， 用来管理多个站点，一个Service最多只能有一个Engine

engine在tomcat中的实现类是：StandardEngine

**host**：engine容器的子容器，一个host对应一个网络域名，一个host包含多个context

host在tomcat中的实现类是：StandardHost

**context**：host容器的子容器，表示一个web应用

context在tomcat中的实现类是：StandardContext

**wrapper**：tomcat中最小的容器单元，表示web应用中的servlet

wrapper在tomcat中的实现类是：StandardWrapper



其中组件中最核心的是Connector 和 Container；Connector负责 请求和响应，Container 负责 处理请求 两者交互如下：



![img](http://pcc.huitogo.club/5dcf290e5aa65422ed2687f06ff4001c)



#### 1. Connector

connector架构：最底层使用的是Socket进行连接的，Request和Response是按照Http协议来封装的，所以Connector同时需要实现TCP/IP协议和Http协议

Connector中具体用事件处理器来处理请求【ProtocoHandler】，不同的ProtocoHandler代表不同的连接类型【所以一个Service中可以有多个Connector】 例如：Http11Protocol使用普通的Socket来连接的，Http11NioProtocol使用NioSocket连接。 Endpoint用于处理底层Socket的网络连接，用来实现Tcp/Ip协议。

Acceptor:用于监听和接收请求。

Handler：请求的初步处理，并且调用Processor

AsynTimeout:检查异步的Request请求是否超时 Processor用于将Endpoint接收Socket封装成Request，用来实现HTTP协议

Adapter 用于将Request交给Container 进行具体处理，即将请求适配到Servlet容器



#### 2. Container

![img](http://pcc.huitogo.club/729b44dc8f59d6aadf5efc9465e472ac)



Container 架构：Container就是一个Engine。Container用于封装和管理Servlet，以及具体处理Request请求