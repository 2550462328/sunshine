#### 1. 什么是Beats？

Beats在是一个轻量级日志采集器，其实Beats家族有6个成员，早期的ELK架构中使用Logstash收集、解析日志，但是Logstash对内存、cpu、io等资源消耗比较高。相比Logstash，Beats所占系统的CPU和内存几乎可以忽略不计。 



目前Beats包含六种工具：

1. Packetbeat：网络数据（收集网络流量数据）
2. Metricbeat：指标（收集系统、进程和文件系统级别的CPU和内存使用情况等数据）
3. Filebeat：日志文件（收集文件数据）
4. Winlogbeat：windows事件日志（收集Windows事件日志数据）
5. Auditbeat：审计数据（收集审计日志）
6. Heartbeat：运行时间监控（收集系统运行时的数据）



#### 2. 什么是fileBeat？

首先filebeat是Beats中的一员。

Filebeat是**用于转发和集中日志数据的轻量级传送工具**。Filebeat监视您指定的日志文件或位置，收集日志事件，并将它们转发到Elasticsearch或 Logstash进行索引。



Filebeat的工作方式如下：启动Filebeat时，它将启动一个或多个输入，这些输入将在为日志数据指定的位置中查找。对于Filebeat所找到的每个日志，Filebeat都会启动收集器。每个收集器都读取单个日志以获取新内容，并将新日志数据发送到libbeat，libbeat将聚集事件，并将聚集的数据发送到为Filebeat配置的输出。



工作的流程图如下：

![img](http://pcc.huitogo.club/ae06381a6391542f8c634a68d4a1ffcb)



#### 3. fileBeat和logstash的关系？

logstash和filebeat都是可以作为日志采集的工具，目前日志采集的工具有很多种，如fluentd, flume, logstash,betas等等。

logstash出现时间要比filebeat早许多，随着时间发展，logstash不仅仅是一个日志采集工具，它也是可以作为一个日志搜集工具，有丰富的input|filter|output插件可以使用。常用的ELK日志采集方案中，大部分的做法就是将所有节点的日志内容上送到kafka消息队列，然后使用logstash集群读取消息队列内容，根据配置文件进行过滤。上送到elasticsearch。

logstash是使用Java编写，插件是使用jruby编写，对机器的资源要求会比较高，网上有一篇关于其性能测试的报告。之前自己也做过和filebeat的测试对比。在采集日志方面，对CPU，内存上都要比前者高很多。

filebeat也是elastic.公司开发的，其官方的说法是为了替代logstash-forward。采用go语言开发。代码开源。



总结来说：**filebeat 是替代 Logstash Forwarder 的下一代 Logstash 收集器。**

![img](http://pcc.huitogo.club/6c42a7950c3eb201680fa9cc6362a710)



#### 4. fileBeat的原理是什么？

##### 4.1 filebeat的构成

filebeat结构：由两个组件构成，分别是inputs（输入）和harvesters（收集器），这些组件一起工作来跟踪文件并将事件数据发送到您指定的输出，harvester负责读取单个文件的内容。harvester逐行读取每个文件，并将内容发送到输出。为每个文件启动一个harvester。harvester负责打开和关闭文件，这意味着文件描述符在harvester运行时保持打开状态。如果在收集文件时删除或重命名文件，Filebeat将继续读取该文件。这样做的副作用是，磁盘上的空间一直保留到harvester关闭。默认情况下，Filebeat保持文件打开，直到达到close_inactive



关闭harvester可以会产生的结果：

文件处理程序关闭，如果harvester仍在读取文件时被删除，则释放底层资源。只有在scan_frequency结束之后，才会再次启动文件的收集。如果该文件在harvester关闭时被移动或删除，该文件的收集将不会继续



一个input负责管理harvesters和寻找所有来源读取。如果input类型是log，则input将查找驱动器上与定义的路径匹配的所有文件，并为每个文件启动一个harvester。每个input在它自己的Go进程中运行，Filebeat当前支持多种输入类型。每个输入类型可以定义多次。日志输入检查每个文件，以查看是否需要启动harvester、是否已经在运行harvester或是否可以忽略该文件。



##### 4.2 filebeat如何保存文件的状态

Filebeat保留每个文件的状态，并经常将状态刷新到磁盘中的注册表文件中。该状态用于记住harvester读取的最后一个偏移量，并确保发送所有日志行。如果无法访问输出（如Elasticsearch或Logstash），Filebeat将跟踪最后发送的行，并在输出再次可用时继续读取文件。当Filebeat运行时，每个输入的状态信息也保存在内存中。当Filebeat重新启动时，来自注册表文件的数据用于重建状态，Filebeat在最后一个已知位置继续每个harvester。对于每个输入，Filebeat都会保留它找到的每个文件的状态。由于文件可以重命名或移动，文件名和路径不足以标识文件。对于每个文件，Filebeat存储唯一的标识符，以检测文件是否以前被捕获。



##### 4.3 filebeat如何保证至少一次数据消费

Filebeat保证事件将至少传递到配置的输出一次，并且不会丢失数据。是因为它将每个事件的传递状态存储在注册表文件中。在已定义的输出被阻止且未确认所有事件的情况下，Filebeat将继续尝试发送事件，直到输出确认已接收到事件为止。如果Filebeat在发送事件的过程中关闭，它不会等待输出确认所有事件后再关闭。当Filebeat重新启动时，将再次将Filebeat关闭前未确认的所有事件发送到输出。这样可以确保每个事件至少发送一次，但最终可能会有重复的事件发送到输出。通过设置shutdown_timeout选项，可以将Filebeat配置为在关机前等待特定时间。