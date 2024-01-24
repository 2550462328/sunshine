#### 1. JMX使用场景

1. 用来管理应用程序的配置项，可以在运行期动态改变配置项的值，而不用妨碍程序的运行，这对与许多可靠性要求较高的应用来说非常方便。

   可以通过jconsole等JMX客户端工具动态改变配置项的值。

2. 用来对应用程序的运行状态进行监控，比如对一个大型交易处理程序，我们要监控当前有多少交易在排队中，每笔交易的处理时间是多少，平均每处理一笔交易要花多少时间等等。

   中间件软件WebLogic的管理页面就是基于JMX开发的，而JBoss则整个系统都基于JMX构架。



对于一些参数的修改，网上有一段描述还是比较形象的：

1. 程序初哥一般是写死在程序中，到要改变的时候就去修改代码，然后重新编译发布。
2. 程序熟手则配置在文件中（JAVA一般都是properties文件），到要改变的时候只要修改配置文件，但还是必须重启系统，以便读取配置文件里最新的值。
3. 程序好手则会写一段代码，把配置值缓存起来，系统在获取的时候，先看看配置文件有没有改动，如有改动则重新从配置里读取，否则从缓存里读取。
4. 程序高手则懂得物为我所用，用JMX把需要配置的属性集中在一个类中，然后写一个MBean，再进行相关配置。另外JMX还提供了一个工具页，以方便我们对参数值进行修改。



#### 2. JMX基础架构

![img](http://pcc.huitogo.club/c4803104ca8e5ad38d46aedec273e517)



其中设备层主要定义了信息模型。在JMX中，各种管理对象以管理构件的形式存在，需要管理时，向MBean服务器进行注册。



MBean 可以看作是JavaBean的一种特殊形式，其定义是符合JavaBean的规范的，它代表 JMX 中的一种可以被管理的资源。MBean会通过接口定义，给出这些资源的一些特定操作：

- 属性的读和写操作；
- 可以被执行的操作；
- 关于自己的描述信息



#### 3. MBean 和 MXBean

接口命名规范分为 **MBean 和 MXBean**

不同之处在于 在MXBean中，如果一个MXBean的接口定义了一个属性是一个自定义类型，如MemoryMXBean中定义了heapMemoryUsage属性，这个属性是MemoryUsage类型的，当JMX使用这个MXBean时，这个MemoryUsage就会被转换成一种标准的类型，这些类型被称为开放类型，是定义在 javax.management.openmbean包中的。而这个转换的规则是，如果是原生类型，如int或者是String，则不会有变化，但如果是其他自定义类型，则被转换成CompositeDataSupport类。

```
// 创建MBeanServer   
MBeanServer mbs = ManagementFactory.getPlatformMBeanServer();   
           
// 新建MBean ObjectName, 在MBeanServer里标识注册的MBean   
ObjectName name = new ObjectName("com.haitao.jmx:type=Echo");   
           
// 创建MBean   
Echo mbean = new Echo();   
           
// 在MBeanServer里注册MBean, 标识为ObjectName(com.tenpay.jmx:type=Echo)   
mbs.registerMBean(mbean, name);   
             
// 在MBeanServer里调用已注册的EchoMBean的print方法   
mbs.invoke(name, "print", new Object[] { "haitao.tu"}, new String[] {"java.lang.String"});     
```