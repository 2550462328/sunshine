Watcher（事件监听器），是 ZooKeeper 中的一个很重要的特性。ZooKeeper 允许用户在指定节点上注册一些 Watcher，并且在一些特定事件触发的时候，ZooKeeper 服务端会将事件通知到感兴趣的客户端上去，该机制是 ZooKeeper 实现分布式协调服务的重要特性。

![img](http://pcc.huitogo.club/de73b1994ef5f43445f72852f7d8d2eb)



Watch 有如下特点：

1. **主动推送**：Watch被触发时，由 ZooKeeper 服务器主动将更新推送给客户端，而不需要客户端轮询。

2. **一次性**：数据变化时，Watch 只会被触发一次。如果客户端想得到后续更新的通知，必须要在 Watch 被触发后重新注册一个 Watch。

3. 轻量级 Watch 机制。

   - Watcher 通知非常简单，只会告诉客户端发生了事件，而不会说明事件的具体内容。
   - 客户端向服务端注册 Watcher 的时候，并不会把客户端真实的 Watcher 对象实体传递到服务端，仅仅是在客户端请求中使用`boolean` 类型属性进行了标记。

4. **可见性**：如果一个客户端在读请求中附带 Watch，Watch 被触发的同时再次读取数据，客户端在得到 Watch 消息之前肯定不可能看到更新后的数据。换句话说，更新通知先于更新结果。

5. **顺序性**：如果多个更新触发了多个 Watch ，那 Watch 被触发的顺序与更新顺序一致。

   Watcher event 异步发送 Watcher 的通知事件从 Server 发送到Client 是异步的，这就存在一个问题，不同的客户端和服务器之间通过Socket 进行通信，由于网络延迟或其他因素导致客户端在不通的时刻监听到事件，由于 Zookeeper 本身提供了 ordering guarantee ，即客户端监听事件后，才会感知它所监视 znode 发生了变化。所以我们使用 Zookeeper 不能期望能够监控到节点每次的变化。**Zookeeper 只能保证最终的一致性，而无法保证强一致性**。



**Watch是怎么实现的？**

**第一步，客户端注册 Watcher 实现？**

1. 调用 getData、getChildren、exist 三个 API ，传入Watcher 对象。
2. 标记请求 request ，封装 Watcher 到 WatchRegistration 。
3. 封装成 Packe t对象，发服务端发送 request 。
4. 收到服务端响应后，将 Watcher 注册到 ZKWatcherManager 中进行管理。
5. 请求返回，完成注册。



 **第二步，服务端处理 Watcher 实现？**

- 服务端接收 Watcher 并存储。

  > 接收到客户端请求，处理请求判断是否需要注册 Watcher ，需要的话将数据节点的节点路径和 ServerCnxn(ServerCnxn 代表一个客户端和服务端的连接，实现了 Watcher 的 process 接口，此时可以看成一个 Watcher 对象)存储在 WatcherManager 的 WatchTable 和 Watch2Paths 中去。

- Watcher 触发。

  > 以服务端接收到 setData 事务请求触发 NodeDataChanged 事件为例：
  >
  > - 封装 WatchedEvent ：
  >
  >   > 将通知状态（SyncConnected）、事件类型（NodeDataChanged）以及节点路径封装成一个WatchedEvent对象
  >
  > - 查询 Watcher ：
  >
  >   > 从 WatchTable 中根据节点路径查找 Watcher 。
  >   >
  >   > - 没找到 ：说明没有客户端在该数据节点上注册过 Watcher 。
  >   > - 找到 ：提取并从 WatchTable 和 Watch2Paths 中删除对应 Watcher (**从这里可以看出 Watcher 在服务端是一次性的，触发一次就失效了**)。

- 调用 process 方法来触发 Watcher 。

  > 这里 process 主要就是通过 ServerCnxn 对应的 TCP 连接发送 Watcher 事件通知。



**第三步，客户端回调 Watcher 实现？**

客户端 SendThread 线程接收事件通知，交由 EventThread 线程回调Watcher 。

客户端的 Watcher 机制同样是一次性的，一旦被触发后，该 Watcher 就失效了。