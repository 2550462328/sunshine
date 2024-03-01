nacos 的整体架构如下

![img](http://pcc.huitogo.club/bf32f9382b38f1e17bc0f3510592ef22)



整体分成 Naming Service (注册和发现服务信息) 和 Config Service(注册 和 发现全局配置)



下图是根据源码整理出的 Naming Service 和 Config Service 基于 Client/Server 交互的流程图

![img](http://pcc.huitogo.club/b4d3b52b2b96feb618bc0a37760fba4c)



从源码的角度来看，无论Naming Service 还是 Config Service 都是 基于 数据存取 和 事件通知角度来跟踪的，其中数据存取就是 Service 和 Config的存取，而事件通知包括

- Nacos Server 集群之间的同步事件
- 监听 Nacos Server 的 Client端的 同步事件