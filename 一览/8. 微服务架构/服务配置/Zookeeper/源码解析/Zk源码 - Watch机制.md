#### 1. watch的数据存储

watcher的关联path存储在WatchManager中的两个HashMap中，watchTable 是 path 和 watcher的一对多关系，watch2Paths 是 watcher和path的一对多关系，两者在不同的场景下搭配使用

```
private final HashMap<String, HashSet<Watcher>> watchTable =
    new HashMap<String, HashSet<Watcher>>();

private final HashMap<Watcher, HashSet<String>> watch2Paths =
    new HashMap<Watcher, HashSet<String>>();
```



添加Watcher的核心方法见WatcherManager.addWatch方法，核心也就分别向watchTable 和 watch2Paths中填充内容

```
synchronized void addWatch(String path, Watcher watcher) {
    HashSet<Watcher> list = watchTable.get(path);
    if (list == null) {
        // don't waste memory if there are few watches on a node
        // rehash when the 4th entry is added, doubling size thereafter
        // seems like a good compromise
        list = new HashSet<Watcher>(4);
        watchTable.put(path, list);
    }
    list.add(watcher);

    HashSet<String> paths = watch2Paths.get(watcher);
    if (paths == null) {
        // cnxns typically have many watches, so use default cap here
        paths = new HashSet<String>();
        watch2Paths.put(watcher, paths);
    }
    paths.add(path);
}
```



添加Watcher的场景有很多种，常见的比如 获取节点数据（getData）、获取节点状态（statNode）、获取子节点（getChildern） 和 直接对节点添加Watch，除直接对节点添加Watch行为外，其他操作均有客户端有选择性的发起，比如我希望实时获取节点数据，即节点数据能在修改的时候通知到我，这一点可以用作 全局配置项 或者 服务发现 的场景中；

直接添加Watch的行为 我们常见于分布式锁，分布式锁会在zk上创建临时有序节点（EPHEMERAL_SEQUENTIAL），当客户端断开连接时节点会自动删除（放弃锁），较大的临时节点存在时（锁占有）可以对该节点进行Watch，该临时节点主动或被动删除后会通知到我（获取锁），这个可以实现公平锁；非公平锁 即直接对父节点进行 Watch，当子节点发生Delete操作时获取到通知 进行抢占锁；当然在并发的情况下由zk保证并发操作的有序性 和 线程安全；



看下的相关代码，以获取节点数据为例

```
case OpCode.getData: {
    lastOp = "GETD";
    GetDataRequest getDataRequest = new GetDataRequest();
    // 将客户端request里的请求 内容 赋给 GetDataRequest
    ByteBufferInputStream.byteBuffer2Record(request.request,
            getDataRequest);
    DataNode n = zks.getZKDatabase().getNode(getDataRequest.getPath());
    if (n == null) {
        throw new KeeperException.NoNodeException();
    }
    PrepRequestProcessor.checkACL(zks, zks.getZKDatabase().aclForNode(n),
            ZooDefs.Perms.READ,
            request.authInfo);
    Stat stat = new Stat();
    // 加入 客户端请求中带有watch参数，则将当前的serer连接视为watcher进行注册
    byte b[] = zks.getZKDatabase().getData(getDataRequest.getPath(), stat,
            getDataRequest.getWatch() ? cnxn : null);
    rsp = new GetDataResponse(b, stat);
    break;
```



这里的cnxn代表当前连接zkServer的连接信息，如果是NIO模式下就是一条Select socket连接

```
private void readRequest() throws IOException {
    // NIO模式下 this代指NIOServerCnxn
    zkServer.processPacket(this, incomingBuffer);
}
```



#### 2. 哪些情况下会触发watch？

触发watch的场景大致分成两种，节点自身变化 和 节点数据变化，分别对应DataTree中的childWatches 和 dataWatches

```
// 关心节点数据变化
private final WatchManager dataWatches = new WatchManager();

// 关心节点自身变化
private final WatchManager childWatches = new WatchManager();
```



正常watch事件触发是在内存中的DataTree修改完成后发生。以删除节点为例，见DataTree.deleteNode()

```
public void deleteNode(String path, long zxid)
        throws KeeperException.NoNodeException {
    //...不重要
    // 数据变化 触发DataWatch的process方法
    Set<Watcher> processed = dataWatches.triggerWatch(path,EventType.NodeDeleted);
    // 节点变化 触发ChildWatch的process方法，这里剔除了数据变化触发的Watcher
    childWatches.triggerWatch(path, EventType.NodeDeleted, processed);
    childWatches.triggerWatch("".equals(parentName) ? "/" : parentName,EventType.NodeChildrenChanged);
}
```



triggerWatch即查询 对节点感兴趣的Watchers，发一条短信（WatchedEvent）通知他们，通知的内容仅包括 通知状态、事件类型 和 节点path

```
public class WatchedEvent {
    // 通知状态
    final private KeeperState keeperState;
    // 事件类型
    final private EventType eventType;
    // 节点path
    private String path;
}

Set<Watcher> triggerWatch(String path, EventType type, Set<Watcher> supress) {
    WatchedEvent e = new WatchedEvent(type,
            KeeperState.SyncConnected, path);
    HashSet<Watcher> watchers;
    synchronized (this) {
        watchers = watchTable.remove(path);
        if (watchers == null || watchers.isEmpty()) {
            if (LOG.isTraceEnabled()) {
                ZooTrace.logTraceMessage(LOG,
                        ZooTrace.EVENT_DELIVERY_TRACE_MASK,
                        "No watchers for " + path);
            }
            return null;
        }
        for (Watcher w : watchers) {
            HashSet<String> paths = watch2Paths.get(w);
            if (paths != null) {
                paths.remove(path);
            }
        }
    }
    for (Watcher w : watchers) {
        if (supress != null && supress.contains(w)) {
            continue;
        }
        w.process(e);
    }
    return watchers;
}
```



#### 3. 触发watch后怎么反馈到监听的客户端？

这里以NIO模式下的服务端为例，对应Watcher（NIOServerCnxn）的WatchEvent通知行为

```
public void process(WatchedEvent event) {
    ReplyHeader h = new ReplyHeader(-1, -1L, 0);
    if (LOG.isTraceEnabled()) {
        ZooTrace.logTraceMessage(LOG, ZooTrace.EVENT_DELIVERY_TRACE_MASK,
                                 "Deliver event " + event + " to 0x"
                                 + Long.toHexString(this.sessionId)
                                 + " through " + this);
    }

    // Convert WatchedEvent to a type that can be sent over the wire
    WatcherEvent e = event.getWrapper();

    sendResponse(h, e, "notification");
}

sendResponse会将响应信息放到outgoingBuffer缓冲队列中
private final Queue<ByteBuffer> outgoingBuffers =
    new LinkedBlockingQueue<ByteBuffer>();

/**
 * sendBuffer pushes a byte buffer onto the outgoing buffer queue for
 * asynchronous writes.
 */
public void sendBuffer(ByteBuffer bb) {
    if (LOG.isTraceEnabled()) {
        LOG.trace("Add a buffer to outgoingBuffers, sk " + sk
                  + " is valid: " + sk.isValid());
    }
    outgoingBuffers.add(bb);
    // 有响应信息了，唤醒selector线程，让它工作（假如不在工作）
    requestInterestOpsUpdate();
}
```



接下的事情就交给Selector线程 和 Worker线程了，Selector线程select出内容后 交给Worker线程处理，Worker 会将outgoingBuffer的内容进行输出