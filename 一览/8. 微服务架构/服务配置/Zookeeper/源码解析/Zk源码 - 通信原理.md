#### 1. zk的启动

![img](http://pcc.huitogo.club/ec2d54ec357814640817a5414b1964ba)



重点看一下ZookeeperServerMain的main方法，主要包含三大块

```
AdminServer.start()   

	DummyAdminServer：什么都不做
	JettyAdminServer： 内嵌Servlet容器，用于访问ZookeeperServer支持的接口（命令行）

ServerCnxnFactory.startup(ZookeeperServer) 初始化 和 客户端之间的连接处理，分 加密模式 和 非加密模式

	NettyServerCnxnFactory：和客户端之间使用Netty传输
	NIOServerCnxnFactory：使用Http的NIO模式进行传输
	
		 start工作线程：
		 
				1 accept thread：接收新的连接 并委派给Selector Thread
				1-N selector threads: 实际select 请求的线程，哪个channel有流传输则进行select，支持多线程用于大量客户端发起请求的场景，避免selector thread变成性能瓶颈
				0-M socket I/O worker threads：实际工作线程，执行RequestProcessor，如果配置为0的话则直接在selector thread中执行
				1 connection expiration thread：根据会话时间（心跳）关闭空闲的客户端连接
				
		装配Zookeeper 服务端配置信息
				startdata()：加载磁盘数据（dataTree）到内存中，并获取最后一次有效的 zxid（节点序列号）
				startup()
						startSessionTracker()： 启动session会话跟踪管理器，leader 和 standalone模式下的 共用一个，其他的follow 服务 则关联到 leader上
						setupRequestProcessors()：配置请求链式处理器  PrepRequestProcessor -> SyncRequestProcessor -> FinalRequestProcessor
						registerJMX()：注册了一个 Zk的服务信息 和 数据信息

ContainerManager.start()：管理 和 清理 zk的数据节点
		checkContainers()：清理无子节点 和 ttl到期的叶子节点
```



简单的总结就是Zookeeper本质上就是管理它的数据节点（DataTree）

详细看下各子部分的代码



NIOServerCnxnFactory.start() 初始化 和 客户端之间的连接处理

```
public void start() {
    stopped = false;
    // 真正执行客户端的请求
    if (workerPool == null) {
        workerPool = new WorkerService(
                "NIOWorker", numWorkerThreads, false);
    }
    // epoll 模型
    // 都是damon 线程
    // 处理 和 验证请求，更新客户端连接状态，主要是讲请求体 转交给 Worker去执行
    for (SelectorThread thread : selectorThreads) {
        if (thread.getState() == Thread.State.NEW) {
            thread.start();
        }
    }
    // ensure thread is started once and only once
    // acceptThread 将 连接请求 ClientSocketChannerl加入 acceptQueue中
    if (acceptThread.getState() == Thread.State.NEW) {
        acceptThread.start();
    }
    // 关闭 过期的（失活）的客户端连接
    if (expirerThread.getState() == Thread.State.NEW) {
        expirerThread.start();
    }
}
```



可以看出这是一块典型的NIO的epoll模式设计，一个Accept Thread拖着多个Selector Thread，然后Selector Thread将读取的客户端连接通道的流信息报文请求分发到WorkPool中 等待Worker执行，最后是一个Exipirer Thread负责 心跳检查



Accept Thread

```
private boolean doAccept() {
    boolean accepted = false;
    SocketChannel sc = null;
    try {
        sc = acceptSocket.accept();
        accepted = true;
        InetAddress ia = sc.socket().getInetAddress();
        int cnxncount = getClientCnxnCount(ia);

        if (maxClientCnxns > 0 && cnxncount >= maxClientCnxns) {
            throw new IOException("Too many connections from " + ia
                    + " - max is " + maxClientCnxns);
        }

        LOG.debug("Accepted socket connection from "
                + sc.socket().getRemoteSocketAddress());
        sc.configureBlocking(false);

        // Round-robin assign this connection to a selector thread
        if (!selectorIterator.hasNext()) {
            selectorIterator = selectorThreads.iterator();
        }
        SelectorThread selectorThread = selectorIterator.next();
        if (!selectorThread.addAcceptedConnection(sc)) {
            throw new IOException(
                    "Unable to add connection to selector queue"
                            + (stopped ? " (shutdown in progress)" : ""));
        }
        acceptErrorLogger.flush();
    } catch (IOException e) {
        // accept, maxClientCnxns, configureBlocking
        acceptErrorLogger.rateLimitLog(
                "Error accepting new connection: " + e.getMessage());
        fastCloseSock(sc);
    }
    return accepted;
}
```



Selector Thread

```
private void select() {
    try {
        selector.select();

        Set<SelectionKey> selected = selector.selectedKeys();
        ArrayList<SelectionKey> selectedList =
                new ArrayList<SelectionKey>(selected);
        Collections.shuffle(selectedList);
        Iterator<SelectionKey> selectedKeys = selectedList.iterator();
        while (!stopped && selectedKeys.hasNext()) {
            SelectionKey key = selectedKeys.next();
            selected.remove(key);

            if (!key.isValid()) {
                cleanupSelectionKey(key);
                continue;
            }
            if (key.isReadable() || key.isWritable()) {
                handleIO(key);
            } else {
                LOG.warn("Unexpected ops in select " + key.readyOps());
            }
        }
    } catch (IOException e) {
        LOG.warn("Ignoring IOException while selecting", e);
    }
}
```



Expire Thread

```
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
```



NIOServerCnxnFactory.startdata() ：加载磁盘数据（dataTree）到内存中，并获取最后一次有效的 zxid（节点序列号）

```
/**
 *  Restore sessions and data
 */
public void loadData() throws IOException, InterruptedException {
    /*
     * When a new leader starts executing Leader#lead, it 
     * invokes this method. The database, however, has been
     * initialized before running leader election so that
     * the server could pick its zxid for its initial vote.
     * It does it by invoking QuorumPeer#getLastLoggedZxid.
     * Consequently, we don't need to initialize it once more
     * and avoid the penalty of loading it a second time. Not 
     * reloading it is particularly important for applications
     * that host a large database.
     * 
     * The following if block checks whether the database has
     * been initialized or not. Note that this method is
     * invoked by at least one other method: 
     * ZooKeeperServer#startdata.
     *  
     * See ZOOKEEPER-1642 for more detail.
     */
    if(zkDb.isInitialized()){
        setZxid(zkDb.getDataTreeLastProcessedZxid());
    }
    else {
        setZxid(zkDb.loadDataBase());
    }
    
    // Clean up dead sessions
    LinkedList<Long> deadSessions = new LinkedList<Long>();
    for (Long session : zkDb.getSessions()) {
        if (zkDb.getSessionWithTimeOuts().get(session) == null) {
            deadSessions.add(session);
        }
    }

    for (long session : deadSessions) {
        // XXX: Is lastProcessedZxid really the best thing to use?
        killSession(session, zkDb.getDataTreeLastProcessedZxid());
    }

    // Make a clean snapshot
    takeSnapshot();
}
```



总结下就是 如果内存有的话直接获取最近一次的zxid，没有则从磁盘（zkdb）中获取，可以是本地文件 也可以是数据库，硬盘存储是以log的形式存储的，映射到内存中的快照则是DataTree结构



NIOServerCnxnFactory.startup()

```
public synchronized void startup() {
    if (sessionTracker == null) {
        createSessionTracker();
    }
    // 启动session会话跟踪管理器，leader 和 standalone模式下的 共用一个，其他的follow 服务 则关联到 leader上
    // 处理主从之间的 会话，清除过期会话
    startSessionTracker();
    // 启动请求链式处理器  PrepRequestProcessor -> SyncRequestProcessor -> FinalRequestProcessor
    // 有点类似于Netty 的 Processor模式
    setupRequestProcessors();

    // 通常使用JMX来监控系统的运行状态或管理系统的某些方面，比如清空缓存、重新加载配置文件等
    // 注册了一个 ZooKeeperServerBean 和 DataTreeBean
    // ZooKeeperServerBean : zk的服务信息
    // DataTreeBean： zk的内存数据信息，DataTree，一个hashtable保存数据信息，用于存取，一个Tree 用于磁盘序列化（快照）
    registerJMX();

    setState(State.RUNNING);
    notifyAll();
}
```



setupRequestProcessors()

```
protected void setupRequestProcessors() {
    RequestProcessor finalProcessor = new FinalRequestProcessor(this);
    RequestProcessor syncProcessor = new SyncRequestProcessor(this,
            finalProcessor);
    ((SyncRequestProcessor)syncProcessor).start();
    firstProcessor = new PrepRequestProcessor(this, syncProcessor);
    ((PrepRequestProcessor)firstProcessor).start();
}
```



可以看出请求处理链的先后顺序是PrepRequestProcessor -> SyncRequestProcessor -> FinalRequestProcessor，跟Netty里面配置的SocketProcessor有点像，比如先读取字符流转码 -> 请求处理 -> 输出字符流编码



ContainerManager.start() 管理 和 清理 zk的数据节点

```
/**
 * Manually check the containers. Not normally used directly
 */
public void checkContainers()
        throws InterruptedException {
    long minIntervalMs = getMinIntervalMs();
    for (String containerPath : getCandidates()) {
        long startMs = Time.currentElapsedTime();

        ByteBuffer path = ByteBuffer.wrap(containerPath.getBytes());
        Request request = new Request(null, 0, 0,
                ZooDefs.OpCode.deleteContainer, path, null);
        try {
            LOG.info("Attempting to delete candidate container: {}",
                    containerPath);
            requestProcessor.processRequest(request);
        } catch (Exception e) {
            LOG.error("Could not delete container: {}",
                    containerPath, e);
        }

        long elapsedMs = Time.currentElapsedTime() - startMs;
        long waitMs = minIntervalMs - elapsedMs;
        if (waitMs > 0) {
            Thread.sleep(waitMs);
        }
    }
}
```



就是清理无效的 数据节点，这里主要看下哪些是无效的数据节点



getCandidates()

```
protected Collection<String> getCandidates() {
    Set<String> candidates = new HashSet<String>();
    for (String containerPath : zkDb.getDataTree().getContainers()) {
        DataNode node = zkDb.getDataTree().getNode(containerPath);
        /*
            cversion > 0: keep newly created containers from being deleted
            before any children have been added. If you were to create the
            container just before a container cleaning period the container
            would be immediately be deleted.
         */
        if ((node != null) && (node.stat.getCversion() > 0) &&
                (node.getChildren().size() == 0)) {
            candidates.add(containerPath);
        }
    }
    for (String ttlPath : zkDb.getDataTree().getTtls()) {
        DataNode node = zkDb.getDataTree().getNode(ttlPath);
        if (node != null) {
            Set<String> children = node.getChildren();
            if ((children == null) || (children.size() == 0)) {
                if ( EphemeralType.get(node.stat.getEphemeralOwner()) == EphemeralType.TTL ) {
                    long elapsed = getElapsed(node);
                    long ttl = EphemeralType.TTL.getValue(node.stat.getEphemeralOwner());
                    if ((ttl != 0) && (getElapsed(node) > ttl)) {
                        candidates.add(ttlPath);
                    }
                }
            }
        }
    }
    return candidates;
}
```



无效的叶子节点即 没有子节点的节点 或者 ttl到期的 叶子节点



#### 2. zkclient 和 zkserver 的请求机制



这里重点看下 Worker的工作范围

```
/**
 * Schedule work to be done by the thread assigned to this id. Thread
 * assignment is a single mod operation on the number of threads.  If a
 * worker thread pool is not being used, work is done directly by
 * this thread.
 */
public void schedule(WorkRequest workRequest, long id) {
    if (stopped) {
        workRequest.cleanup();
        return;
    }

    ScheduledWorkRequest scheduledWorkRequest =
        new ScheduledWorkRequest(workRequest);

    // If we have a worker thread pool, use that; otherwise, do the work
    // directly.
    int size = workers.size();
    if (size > 0) {
        try {
            // make sure to map negative ids as well to [0, size-1]|
            // 随机挑选一个幸运工人，线程池的最大数 在0~size-1之间
            int workerNum = ((int) (id % size) + size) % size;
            ExecutorService worker = workers.get(workerNum);
            worker.execute(scheduledWorkRequest);
        } catch (RejectedExecutionException e) {
            LOG.warn("ExecutorService rejected execution", e);
            workRequest.cleanup();
        }
    } else {
        // When there is no worker thread pool, do the work directly
        // and wait for its completion
        scheduledWorkRequest.run();
    }
}

// 工人开始工作了
private class ScheduledWorkRequest implements Runnable {
    
    @Override
    public void run() {
        try {
            // Check if stopped while request was on queue
            if (stopped) {
                workRequest.cleanup();
                return;
            }
            workRequest.doWork();
        } catch (Exception e) {
            LOG.warn("Unexpected exception", e);
            workRequest.cleanup();
        }
    }
}
```



从 workRequest.doWork() 可追踪到 zkServer.processPacket(this, incomingBuffer)

```
ZookeeperServer.processPacket(ServerCnxn, ByteBuffer)  处理请求报文
public void processPacket(ServerCnxn cnxn, ByteBuffer incomingBuffer) throws IOException {
    // We have the request, now process and setup for next
    InputStream bais = new ByteBufferInputStream(incomingBuffer);
    BinaryInputArchive bia = BinaryInputArchive.getArchive(bais);
    RequestHeader h = new RequestHeader();
    h.deserialize(bia, "header");
    // Through the magic of byte buffers, txn will not be
    // pointing
    // to the start of the txn
    incomingBuffer = incomingBuffer.slice();
    // 授权认证请求
    if (h.getType() == OpCode.auth) {
       //... 不关心的内容
    } else {
        // ssl请求
        if (h.getType() == OpCode.sasl) {
            // ... 不关心的内容
        } // 正文处理
        else {
            Request si = new Request(cnxn, cnxn.getSessionId(), h.getXid(),
              h.getType(), incomingBuffer, cnxn.getAuthInfo());
            si.setOwner(ServerCnxn.me);
            // Always treat packet from the client as a possible
            // local request.
            setLocalSessionFlag(si);
            // 提交请求
            submitRequest(si);
        }
    }
    cnxn.incrOutstandingRequests(h);
}
```



submitRequest(Request）

```
public void submitRequest(Request si) {
    // zk 服务端正在启动，稍等片刻
    if (firstProcessor == null) {
        synchronized (this) {
            try {
                // Since all requests are passed to the request
                // processor it should wait for setting up the request
                // processor chain. The state will be updated to RUNNING
                // after the setup.
                while (state == State.INITIAL) {
                    wait(1000);
                }
            } catch (InterruptedException e) {
                LOG.warn("Unexpected interruption", e);
            }
            if (firstProcessor == null || state != State.RUNNING) {
                throw new RuntimeException("Not started");
            }
        }
    }
    try {
        // 客户端有请求来了，说明它是活的
        touch(si.cnxn);
        boolean validpacket = Request.isValid(si.type);
        if (validpacket) {
            firstProcessor.processRequest(si);
            if (si.cnxn != null) {
                // 请求数 + 1，因为zk服务端会作一个等待处理的最大请求数量的限制
                incInProcess();
            }
        } else {
            LOG.warn("Received packet at server of unknown type " + si.type);
            new UnimplementedRequestProcessor().processRequest(si);
        }
    } catch (MissingSessionException e) {
        if (LOG.isDebugEnabled()) {
            LOG.debug("Dropping request: " + e.getMessage());
        }
    } catch (RequestProcessorException e) {
        LOG.error("Unable to process request:" + e.getMessage(), e);
    }
}
```



可以看到有我们熟悉的 FirstProcessor，根据我们之前配置的Processor链来看，第一个应该是PrepRequestProcessor

```
LinkedBlockingQueue<Request> submittedRequests = new LinkedBlockingQueue<Request>();

public void processRequest(Request request) {
    submittedRequests.add(request);
}
```

往阻塞队列里面加了一个任务处理，这里用一个队列 缓冲任务，可以实现异步操作



真正的处理在PrepRequestProcessor 的 线程任务中

```
public void run() {
    try {
        while (true) {
            Request request = submittedRequests.take();
            long traceMask = ZooTrace.CLIENT_REQUEST_TRACE_MASK;
            if (request.type == OpCode.ping) {
                traceMask = ZooTrace.CLIENT_PING_TRACE_MASK;
            }
            // 日志追踪，看是客户端请求日志还是心跳包日志
            if (LOG.isTraceEnabled()) {
                ZooTrace.logRequest(LOG, traceMask, 'P', request, "");
            }
            if (Request.requestOfDeath == request) {
                break;
            }
            // 实际执行
            pRequest(request);
        }
}

/**
 * This method will be called inside the ProcessRequestThread, which is a
 * singleton, so there will be a single thread calling this code.
 *
 * @param request
 */
protected void pRequest(Request request) throws RequestProcessorException {
    // LOG.info("Prep>>> cxid = " + request.cxid + " type = " +
    // request.type + " id = 0x" + Long.toHexString(request.sessionId));
    request.setHdr(null);
    request.setTxn(null);

    try {
        switch (request.type) {
        case OpCode.createContainer:
        case OpCode.create:
        case OpCode.create2:
            CreateRequest create2Request = new CreateRequest();
            pRequest2Txn(request.type, zks.getNextZxid(), request, create2Request, true);
            break;
        case OpCode.createTTL:
            CreateTTLRequest createTtlRequest = new CreateTTLRequest();
            pRequest2Txn(request.type, zks.getNextZxid(), request, createTtlRequest, true);
            break;
        case OpCode.deleteContainer:
        case OpCode.delete:
            DeleteRequest deleteRequest = new DeleteRequest();
            pRequest2Txn(request.type, zks.getNextZxid(), request, deleteRequest, true);
            break;
        case OpCode.setData:
            SetDataRequest setDataRequest = new SetDataRequest();                
            pRequest2Txn(request.type, zks.getNextZxid(), request, setDataRequest, true);
            break;
}
```



这里我们以createNode指令为例看看流程怎么执行的

```
private void pRequest2TxnCreate(int type, Request request, Record record, boolean deserialize) throws IOException, KeeperException {
    if (deserialize) {
        ByteBufferInputStream.byteBuffer2Record(request.request, record);
    }

    int flags;
    String path;
    List<ACL> acl;
    byte[] data;
    long ttl;
    if (type == OpCode.createTTL) {
        CreateTTLRequest createTtlRequest = (CreateTTLRequest)record;
        flags = createTtlRequest.getFlags();
        path = createTtlRequest.getPath();
        acl = createTtlRequest.getAcl();
        data = createTtlRequest.getData();
        ttl = createTtlRequest.getTtl();
    } else {
        CreateRequest createRequest = (CreateRequest)record;
        flags = createRequest.getFlags();
        path = createRequest.getPath();
        acl = createRequest.getAcl();
        data = createRequest.getData();
        ttl = -1;
    }
    // 节点类型
    CreateMode createMode = CreateMode.fromFlag(flags);
    validateCreateRequest(path, createMode, request, ttl);
    String parentPath = validatePathForCreate(path, request.sessionId);

    List<ACL> listACL = fixupACL(path, request.authInfo, acl);
    // 我理解的 是修正操作（事务），状态介于准备修改 和 修改完成之间
    ChangeRecord parentRecord = getRecordForPath(parentPath);

    checkACL(zks, parentRecord.acl, ZooDefs.Perms.CREATE, request.authInfo);
    int parentCVersion = parentRecord.stat.getCversion();
    if (createMode.isSequential()) {
        path = path + String.format(Locale.ENGLISH, "%010d", parentCVersion);
    }
    validatePath(path, request.sessionId);
    try {
        // 节点有 修改操作，即不为空
        if (getRecordForPath(path) != null) {
            throw new KeeperException.NodeExistsException(path);
        }
    } catch (KeeperException.NoNodeException e) {
        // ignore this one
    }
    // 判断父节点是否叶子节点
    boolean ephemeralParent = EphemeralType.get(parentRecord.stat.getEphemeralOwner()) == EphemeralType.NORMAL;
    if (ephemeralParent) {
        throw new KeeperException.NoChildrenForEphemeralsException(path);
    }
    // 修改父节点的 版本号
    int newCversion = parentRecord.stat.getCversion() + 1;
    if (type == OpCode.createContainer) {
        request.setTxn(new CreateContainerTxn(path, data, listACL, newCversion));
    } else if (type == OpCode.createTTL) {
        request.setTxn(new CreateTTLTxn(path, data, listACL, newCversion, ttl));
    } else {
        request.setTxn(new CreateTxn(path, data, listACL, createMode.isEphemeral(),
                newCversion));
    }
    StatPersisted s = new StatPersisted();
    if (createMode.isEphemeral()) {
        s.setEphemeralOwner(request.sessionId);
    }
    //写时复制
    parentRecord = parentRecord.duplicate(request.getHdr().getZxid());
    parentRecord.childCount++;
    parentRecord.stat.setCversion(newCversion);
    addChangeRecord(parentRecord);
    addChangeRecord(new ChangeRecord(request.getHdr().getZxid(), path, s, 0, listACL));
}
```



这里在校验完 新增节点 和 新增节点的父节点有效性后 使用 写时复制 的方式 修改父节点的信息（版本号、子节点数量），添加了 新增节点的修改记录，addChangeRecord操作 向事件队列中 添加新增事件信息

```
private void addChangeRecord(ChangeRecord c) {
    synchronized (zks.outstandingChanges) {
        // 事件队列（双端）
        zks.outstandingChanges.add(c);
        // 修改事件内容（非安全的）
        zks.outstandingChangesForPath.put(c.path, c);
    }
}
```



按照我们之前整理的调用链的顺序，下一个是 SyncRequestProcessor，这个处理器是将节点事件生成snapshot 存入磁盘，我们后面再看



最后一个处理器是 FinalRequestProcessor，也就是真正对数据进行操作的地方，看一下 processRequest 方法

```
synchronized (zks.outstandingChanges) {
    // Need to process local session requests
    // 修改本地内存（DataTree）中的节点信息
    rc = zks.processTxn(request);

    // request.hdr is set for write requests, which are the only ones
    // that add to outstandingChanges.
    if (request.getHdr() != null) {
        TxnHeader hdr = request.getHdr();
        Record txn = request.getTxn();
        long zxid = hdr.getZxid();
        while (!zks.outstandingChanges.isEmpty()
               && zks.outstandingChanges.peek().zxid <= zxid) {
            // 移除事务id 小的       
            ChangeRecord cr = zks.outstandingChanges.remove();
            if (cr.zxid < zxid) {
                LOG.warn("Zxid outstanding " + cr.zxid
                         + " is less than current " + zxid);
            }
            // 重复操作同一 path 进行移除，已最新的操作为准
            if (zks.outstandingChangesForPath.get(cr.path) == cr) {
                zks.outstandingChangesForPath.remove(cr.path);
            }
        }
    }

    // do not add non quorum packets to the queue.
    // 新增日志记录（时间轴的形式），用于从节点之间的数据同步
    if (request.isQuorum()) {
        zks.getZKDatabase().addCommittedProposal(request);
    }
}
if (request.cnxn == null) {
    return;
}
ServerCnxn cnxn = request.cnxn;

String lastOp = "NA";
// zk客户端连接数减 1
zks.decInProcess();
Code err = Code.OK;
// 根据事件节点类型 返回 响应报文
Record rsp = null;
...
//审计操作
}

```

对事件缓冲队列进行弹出操作，先写入内存再写入磁盘，最后返回响应报文。