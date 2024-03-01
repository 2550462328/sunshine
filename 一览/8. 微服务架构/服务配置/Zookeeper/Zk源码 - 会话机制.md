#### 1. 客户端连接数限制

先简单总结 最大客户端连接数为60，超过60则拒绝连接；对存在的客户端连接有会话时间限制（默认10s），10s内无任何消息处理（select）则被ExpireThread清理掉；当客户端有消息处理（select）时则给它续命（默认10s）

```
protected int maxClientCnxns = 60;
```



客户端连接数过大时则拒绝连接，被AcceptThread拒绝

```
private boolean doAccept() {
    boolean accepted = false;
    SocketChannel sc = null;
    try {
        sc = acceptSocket.accept();
        accepted = true;
        InetAddress ia = sc.socket().getInetAddress();
        int cnxncount = getClientCnxnCount(ia);
        // 默认为60
        if (maxClientCnxns > 0 && cnxncount >= maxClientCnxns) {
            throw new IOException("Too many connections from " + ia
                    + " - max is " + maxClientCnxns);
        }
//...
}
```



SelectorThread有新客户端连接请求时封装成NIOServerCnxn 进行管理，并关联（attach）到SelectKey上

```
/**
 * Iterate over the queue of accepted connections that have been
 * assigned to this thread but not yet placed on the selector.
 */
private void processAcceptedConnections() {
    SocketChannel accepted;
    while (!stopped && (accepted = acceptedQueue.poll()) != null) {
        // 选择键维护了通道和选择器之间的关联，可以通过选择键获取Channel或Selector，键对象表示一种特殊的关联关系
        SelectionKey key = null;
        try {
            key = accepted.register(selector, SelectionKey.OP_READ);
            NIOServerCnxn cnxn = createConnection(accepted, key, this);
            key.attach(cnxn);
            // 添加进 cnxns中
            addCnxn(cnxn);
        } catch (IOException e) {
            // register, createConnection
            cleanupSelectionKey(key);
            fastCloseSock(accepted);
        }
    }
}
```



ExpireThread清理掉过期会话，关闭客户端连接

```
private class ConnectionExpirerThread extends ZooKeeperThread {

    public void run() {
        try {
            while (!stopped) {
                long waitTime = cnxnExpiryQueue.getWaitTime();
                if (waitTime > 0) {
                    Thread.sleep(waitTime);
                    continue;
                }
                for (NIOServerCnxn conn : cnxnExpiryQueue.poll()) {
                    conn.close();
                }
            }

        } catch (InterruptedException e) {
            LOG.info("ConnnectionExpirerThread interrupted");
        }
    }
}
```



SelectorTrhead有客户端请求时修改客户端的会话时间（状态）

```
/**
 * Schedule I/O for processing on the connection associated with
 * the given SelectionKey. If a worker thread pool is not being used,
 * I/O is run directly by this thread.
 */
private void handleIO(SelectionKey key) {
    IOWorkRequest workRequest = new IOWorkRequest(this, key);
    NIOServerCnxn cnxn = (NIOServerCnxn) key.attachment();

    // Stop selecting this key while processing on its
    // connection
    cnxn.disableSelectable();
    key.interestOps(0);
    //更新客户端的连接状态
    touchCnxn(cnxn);
    workerPool.schedule(workRequest);
}

/**
 * Add or update cnxn in our cnxnExpiryQueue
 * @param cnxn
 */
public void touchCnxn(NIOServerCnxn cnxn) {
    //默认session有效时长为10s
    cnxnExpiryQueue.update(cnxn, cnxn.getSessionTimeout());
}
```



#### 2. 客户端会话跟踪

会话管理 和 上面客户端连接管理的区别在于 出发点不一样，客户端连接管理 是出于服务端性能考虑，关闭不必要 或 空闲的客户端连接，直接从socket层面给它close掉；会话管理是 出于业务安全考虑，更多的偏向于一种权限管理，比如验证会话的有效性，会话是否超时等；两者的共同之处在于都借助于使用ExpiryQueue来清理失效连接，当连接有活跃请求时又会touch连接（续费）。



Session是 ZooKeeper中的会话实体,代表了一个客户端会话。其包含以下4个基本属性。

- sessionID:会话ID,用来唯一标识一个会话,每次客户端创建新会话的时候,ZooKeeper都会为其分配一个全局唯一的 sessionID。
- TimeOut:会话超时时间。客户端在构造 ZooKeeper实例的时候,会配置一个sessiontimeout参数用于指定会话的超时时间。ZooKeeper客户端向服务器发送这个超时时间后,服务器会根据自己的超时时间限制最终确定会话的超时时间。
- TickTime:下次会话超时时间点。为了便于 ZooKeeper对会话实行“分桶策略”管理,同时也是为了高效低耗地实现会话的超时检查与清理, ZooKeeper会为每个会话标记一个下次会话超时时间点。 TickTime是一个13位的long型数据,其值接近于当前时间加上 Time Out,但不完全相等。关于 TickTime的计算方式,将在“分桶策略”部分做详细讲解。
- isClosing:该属性用于标记一个会话是否已经被关闭。通常当服务端检测到一个会话已经超时失效的时候,会将该会话的 isClosing属性标记为“已关闭”,这样就能确保不再处理来自该会话的新请求了。



这里分桶策略就是将 会话 根据 expireTime进行分桶管理，然后Leader服务器在运行期间会定时地对不同的分桶进行会话超时检査。



![img](http://pcc.huitogo.club/7aae462cff2f5041d2bf75e628e6d134)



**session的创建**

服务端对于客户端的“会话创建”请求的处理,大体可以分为四大步骤,分别是处理ConnectRequest请求、会话创建、处理器链路处理和会话响应。在 ZooKeeper服务端,首先将会由NI0 Servercnxn来负责接收来自客户端的“会话创建”请求,并反序列化出ConnectRequest请求,然后根据 ZooKeeper服务端的配置完成会话超时时间的协商。随后, Session tracker将会为该会话分配一个 sessionID,并将其注册到sessionsById和 sessionsWithTimeout中去,同时进行会话的激活。之后,该“会话请求”还会在 ZooKeeper服务端的各个请求处理器之间进行顺序流转,最终完成会话的创建。

```
public void processConnectRequest(ServerCnxn cnxn, ByteBuffer incomingBuffer) throws IOException {
    BinaryInputArchive bia = BinaryInputArchive.getArchive(new ByteBufferInputStream(incomingBuffer));
    ConnectRequest connReq = new ConnectRequest();
    connReq.deserialize(bia, "connect");
    //...
    int sessionTimeout = connReq.getTimeOut();
    byte passwd[] = connReq.getPasswd();
    int minSessionTimeout = getMinSessionTimeout();
    if (sessionTimeout < minSessionTimeout) {
        sessionTimeout = minSessionTimeout;
    }
    int maxSessionTimeout = getMaxSessionTimeout();
    if (sessionTimeout > maxSessionTimeout) {
        sessionTimeout = maxSessionTimeout;
    }
    cnxn.setSessionTimeout(sessionTimeout);
    // We don't want to receive any packets until we are sure that the
    // session is setup
    cnxn.disableRecv();
    long sessionId = connReq.getSessionId();
    if (sessionId == 0) {
        long id = createSession(cnxn, passwd, sessionTimeout);
    } else {
        long clientSessionId = connReq.getSessionId();
        if (serverCnxnFactory != null) {
            serverCnxnFactory.closeSession(sessionId);
        }
        if (secureServerCnxnFactory != null) {
            secureServerCnxnFactory.closeSession(sessionId);
        }
        cnxn.setSessionId(sessionId);
        reopenSession(cnxn, sessionId, passwd, sessionTimeout);
    }
}
```



主要关注sessionId的生成，在客户端没有提供的情况下 且是初始化连接则需要zk服务端自行创建唯一会话id，默认采用的是自增的方式，但自增的初始值是 serverId + 时间戳的组合方式

```
private final AtomicLong nextSessionId = new AtomicLong();

/**
 * Generates an initial sessionId. High order byte is serverId, next 5
 * 5 bytes are from timestamp, and low order 2 bytes are 0s.
 */
public static long initializeNextSession(long id) {
    long nextSid;
    nextSid = (Time.currentElapsedTime() << 24) >>> 8;
    nextSid =  nextSid | (id <<56);
    if (nextSid == EphemeralType.CONTAINER_EPHEMERAL_OWNER) {
        ++nextSid;  // this is an unlikely edge case, but check it just in case
    }
    return nextSid;
}

public long createSession(int sessionTimeout) {
    // 自增
    long sessionId = nextSessionId.getAndIncrement();
    addSession(sessionId, sessionTimeout);
    return sessionId;
}
```



存储session的是两个数据结构，sessionsWithTimeout 和 sessionsById，前者记录session的超时时间，后者记录session的详细信息，添加session如下：

```
public synchronized boolean addSession(long id, int sessionTimeout) {
    sessionsWithTimeout.put(id, sessionTimeout);

    boolean added = false;

    SessionImpl session = sessionsById.get(id);
    if (session == null){
        session = new SessionImpl(id, sessionTimeout);
    }

    // findbugs2.0.3 complains about get after put.
    // long term strategy would be use computeIfAbsent after JDK 1.8
    SessionImpl existedSession = sessionsById.putIfAbsent(id, session);

    if (existedSession != null) {
        session = existedSession;
    } else {
        added = true;
        LOG.debug("Adding session 0x" + Long.toHexString(id));
    }

    if (LOG.isTraceEnabled()) {
        String actionStr = added ? "Adding" : "Existing";
        ZooTrace.logTraceMessage(LOG, ZooTrace.SESSION_TRACE_MASK,
                "SessionTrackerImpl --- " + actionStr + " session 0x"
                + Long.toHexString(id) + " " + sessionTimeout);
    }
    // 激活session（充值）
    updateSessionExpiry(session, sessionTimeout);
    return added;
}
```



这里session激活也是利用ExpiryQueue，同样session失效会话的清理也是类似借助线程完成

```
@Override
public void run() {
    try {
        while (running) {
            long waitTime = sessionExpiryQueue.getWaitTime();
            if (waitTime > 0) {
                Thread.sleep(waitTime);
                continue;
            }

            for (SessionImpl s : sessionExpiryQueue.poll()) {
                setSessionClosing(s.sessionId);
                expirer.expire(s);
            }
        }
    } catch (InterruptedException e) {
        handleException(this.getName(), e);
    }
    LOG.info("SessionTrackerImpl exited loop!");
}
```



session创建完之后会发起一条CreateSession的请求交给处理器链处理

```
long createSession(ServerCnxn cnxn, byte passwd[], int timeout) {
    if (passwd == null) {
        // Possible since it's just deserialized from a packet on the wire.
        passwd = new byte[0];
    }
    // 自增的sessionId
    long sessionId = sessionTracker.createSession(timeout);
    Random r = new Random(sessionId ^ superSecret);
    // 加密密码
    r.nextBytes(passwd);
    ByteBuffer to = ByteBuffer.allocate(4);
    to.putInt(timeout);
    cnxn.setSessionId(sessionId);
    Request si = new Request(cnxn, sessionId, 0, OpCode.createSession, to, null);
    setLocalSessionFlag(si);
    submitRequest(si);
    return sessionId;
}
```



其中预处理器 和 最终处理器均执行的addSession()方法

```
case OpCode.createSession:
    request.request.rewind();
    int to = request.request.getInt();
    request.setTxn(new CreateSessionTxn(to));
    request.request.rewind();
    if (request.isLocalSession()) {
        // This will add to local session tracker if it is enabled
        zks.sessionTracker.addSession(request.sessionId, to);
    } else {
        // Explicitly add to global session if the flag is not set
        zks.sessionTracker.addGlobalSession(request.sessionId, to);
    }
    zks.setOwner(request.sessionId, request.getOwner());
    break;
```



这里我的理解是 为了让session保持最佳有效时长，避免因为连接请求处理时间过长导致会话失效，毕竟客户端发起ping请求也得是 连接建立完之后



request的owner 指的是发起请求来源，这里来源是 客户端请求即ServerCnxn.me，还可能是follwer发起的请求

```
// setowner as the leader itself, unless updated
// via the follower handlers
setOwner(sessionId, ServerCnxn.me);
```



当客户端有发起请求的时候会对重新激活会话

```
public void submitRequest(Request si) {
    //...
    // 客户端有请求来了，激活会话
    touch(si.cnxn);
    //...
}

void touch(ServerCnxn cnxn) throws MissingSessionException {
    if (cnxn == null) {
        return;
    }
    long id = cnxn.getSessionId();
    int to = cnxn.getSessionTimeout();
    if (!sessionTracker.touchSession(id, to)) {
        throw new MissingSessionException(
                "No session with sessionid 0x" + Long.toHexString(id)
                + " exists, probably expired and removed");
    }
}
```



重新激活会话 更新expiryQueue 中会话的下次失效时间，默认3s

```
synchronized public boolean touchSession(long sessionId, int timeout) {
    SessionImpl s = sessionsById.get(sessionId);

    if (s == null) {
        logTraceTouchInvalidSession(sessionId, timeout);
        return false;
    }

    if (s.isClosing()) {
        logTraceTouchClosingSession(sessionId, timeout);
        return false;
    }

    updateSessionExpiry(s, timeout);
    return true;
}
```



会话一个重要的使用场景就是对客户端发起的请求进行校验，比如在预处理器中处理Delete操作之前

```
case OpCode.delete:
    zks.sessionTracker.checkSession(request.sessionId, request.getOwner());
```



校验会话一方面判断sessionId是否存在（存活），另一方面判断请求来源是否正确

```
public synchronized void checkSession(long sessionId, Object owner)
        throws KeeperException.SessionExpiredException,
        KeeperException.SessionMovedException,
        KeeperException.UnknownSessionException {
    LOG.debug("Checking session 0x" + Long.toHexString(sessionId));
    SessionImpl session = sessionsById.get(sessionId);

    if (session == null) {
        throw new KeeperException.UnknownSessionException();
    }

    if (session.isClosing()) {
        throw new KeeperException.SessionExpiredException();
    }

    if (session.owner == null) {
        session.owner = owner;
    } else if (session.owner != owner) {
        throw new KeeperException.SessionMovedException();
    }
}
```