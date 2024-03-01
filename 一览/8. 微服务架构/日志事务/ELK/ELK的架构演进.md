#### 1. 什么是ELK架构？

ELK是软件集合Elasticsearch、Logstash、Kibana的简称，由这三个软件及其相关的组件可以打造大规模日志实时处理系统。

- **Elasticsearch**是实时全文搜索和分析引擎，提供搜集、分析、存储数据三大功能；是一套开放REST和JAVA API等结构提供高效搜索功能，可扩展的分布式系统。它构建于Apache Lucene搜索引擎库之上。
- **Logstash**是一个用来搜集、分析、过滤日志的工具。它支持几乎任何类型的日志，包括系统日志、错误日志和自定义应用程序日志。它可以从许多来源接收日志，这些来源包括 syslog、消息传递（例如 RabbitMQ）和JMX，它能够以多种方式输出数据，包括电子邮件、websockets和Elasticsearch。
- **Kibana**是一个基于Web的图形界面，用于搜索、分析和可视化存储在 Elasticsearch指标中的日志数据。它利用Elasticsearch的REST接口来检索数据，不仅允许用户创建他们自己的数据的定制仪表板视图，还允许他们以特殊的方式查询和过滤数据。



所谓“大规模”，指的是 ELK Stack 组成的系统以一种水平扩展的方式支持每天收集、过滤、索引和存储TB规模以上的各类日志。通常，各类文本形式的日志都在处理范围，包括但不限于 Web 访问日志，如 Nginx/Apache Access Log 。

基于对日志的实时分析，可以随时掌握服务的运行状况、统计 PV/UV、发现异常流量、分析用户行为、查看热门站内搜索关键词等。



#### 2. ELK的架构演进

##### 2.1 架构一

![img](http://pcc.huitogo.club/4c328becd357694d0d458485791d9f4d)



首先由Logstash分布于各个节点上搜集相关日志、数据，并经过分析、过滤后发送给远端服务器上的Elasticsearch进行存储。Elasticsearch将数据以分片的形式压缩存储并提供多种API供用户查询，操作。用户也可以更直观的通过配置Kibana Web Portal方便的对日志查询，并根据数据生成报表。



优点：搭建简单，易于上手。

缺点：**logstash消耗资源大，运行占用cpu和内存高，另外没有消息队列缓存，存在数据丢失隐患。**

应用场景：适合小规模集群使用。



##### 2.2 架构二

![img](http://pcc.huitogo.club/0ddf4ca02828dcc5dd14893519002846)



第二种架构引入了消息队列机制，位于各个节点上的Logstash Agent先将数据/日志传递给Kafka（或者Redis），并将队列中消息或数据间接传递给Logstash，Logstash过滤、分析后将数据传递给Elasticsearch存储。最后由Kibana将日志和数据呈现给用户。因为引入了Kafka（或者Redis）,所以即使远端Logstash server因故障停止运行，数据将会先被存储下来，从而避免数据丢失。



优点：引入了消息队列机制，均衡了网络传输，从而降低了网络闭塞尤其是丢失数据的可能性。

缺点：依然存在Logstash占用系统资源过多的问题。

应用场景：适合较大集群方案。



##### 2.3 架构三

![img](http://pcc.huitogo.club/1a8af91c6b0c5ca1a07aabef60ea5d35)



第三种架构引入了Logstash-forwarder。首先，Logstash-forwarder将日志数据搜集并统一发送给主节点上的Logstash，Logstash分析、过滤日志数据后发送至Elasticsearch存储，并由Kibana最终将数据呈现给用户。



优点：

1. 解决了logstash占用系统资源较高的问题。
2. Logstash-forwarder和Logstash间的通信是通过SSL加密传输，起到了安全保障。



小结：如果是较大集群，用户亦可以如结构三那样配置logstash集群和Elasticsearch集群，引入High Available机制，提高数据传输和存储安全。更主要的配置多个Elasticsearch服务，有助于搜索和数据存储效率。

缺点：Logstash-forwarder和Logstash间通信必须由SSL加密传输，这样便有了一定的限制性。

应用场景：适合较大集群。



##### 2.4 架构四（推荐）

![img](http://pcc.huitogo.club/cabf05c6f415b6940898800a9c38611b)



第四种架构将Logstash-forwarder替换为Beats。经测试，Beats满负荷状态所耗系统资源和Logstash-forwarder相当，但其扩展性和灵活性有很大提高。后面我们会详细介绍一下架构四中的Beats。

这种架构原理基于第三种架构，但是更灵活，扩展性更强。同时可配置Logstash 和Elasticsearch 集群用于支持大集群系统的运维日志数据监控和查询。



总结：不管采用上面哪种ELK架构，都包含了其核心组件，即：Logstash、Elasticsearch 和Kibana。当然这三个组件并非不能被替换，只是就性能和功能性而言，这三个组件已经配合的很完美，是密不可分的。在系统运维中究竟该采用哪种架构，可根据现实情况和架构优劣而定。



#### 3. ELK的功能模块

ELK组件各个功能模块如下图所示，它运行于**分布式系统**之上，通过**搜集**、**过滤**、**传输**、**储存**，**对海量系统和组件日志进行集中管理**和**准实时搜索**、**分析**，使用**搜索**、**监控**、**事件消息**和**报表**等简单易用的功能，帮助运维人员进行**线上业务的准实时监控、业务异常时及时定位原因、排除故障、程序研发时跟踪分析Bug、业务趋势分析、安全与合规审计，深度挖掘日志的大数据价值**。同时Elasticsearch提供多种API（**RESTJAVAPYTHON**等API）供用户扩展开发，以满足其不同需求。

![img](http://pcc.huitogo.club/a0b2e7a5c6c123226b36af93c11b2f20)



#### 4. ELK的使用场景

- 日志查询，问题排查，上线检查。
- 服务器监控，应用监控，错误报警，bug管理。
- 性能分析，用户行为分析，安全漏洞分析，时间管理。