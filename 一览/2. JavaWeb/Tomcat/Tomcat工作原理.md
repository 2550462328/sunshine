这里需要着重说一下 Connector 和 Container的细节

![img](http://pcc.huitogo.club/a8b78a1b823d55f60128420d931605a5)

1. Endpoint接收Socket连接，生成一个SocketProcessor任务提交到线程池去处理
2. SocketProcessor的run方法会调用Processor组件去解析应用层协议，Processor通过解析生成Request对象后，会调 用Adapter的service方法。



其中：

**1. EndPoint**

提供字节流给Processor

监听通信端口，是对传输层的抽象，用来实现 TCP/IP 协议的。 对应的抽象类为AbstractEndPoint，有很多实现类，比如NioEndPoint，JIoEndPoint等。在其中有两个组件，一个 是Acceptor，另外一个是SocketProcessor。 Acceptor用于监听Socket连接请求，SocketProcessor用于处理接收到的Socket请求。



**2. Processor**

提供Tomcat Request对象给Adapter

Processor是用于实现HTTP协议的，也就是说Processor是针对应用层协议的抽象。 Processor接受来自EndPoint的Socket，然后解析成Tomcat Request和Tomcat Response对象，最后通过Adapter 提交给容器。 对应的抽象类为AbstractProcessor，有很多实现类，比如AjpProcessor、Http11Processor等。



**3. Adapter**

提供ServletRequest给容器

ProtocolHandler接口负责解析请求并生成 Tomcat Request 类。 需要把这个 Request 对象转换成 ServletRequest。 Tomcat 引入CoyoteAdapter，这是适配器模式的经典运用，连接器调用 CoyoteAdapter 的 sevice 方法，传入的是 Tomcat Request 对象，CoyoteAdapter 负责将 Tomcat Request 转成 ServletRequest，再调用容器的 service 方 法。



所以 CoyoteAdapter 是 Connector 请求的 处理适配器， 将 HttpRequest 转换成 ServletRequest 交由 Container 处理