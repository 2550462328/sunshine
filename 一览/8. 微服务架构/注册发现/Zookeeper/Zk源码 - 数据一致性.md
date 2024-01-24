**zk怎么保证线程安全 和 数据的一致性**

#### 1. zxid 的原子性

每一个客户端请求会带上xid（事务id）已经服务端会生成全局zxid，执行请求的时候会校验zxid的时效性



客户端发起请求时，Zookeeper服务端会自动生成唯一的递增的zxid 作为事务id，生成途径如下：

```
protected void pRequest(Request request) throws RequestProcessorException 
    try {
        switch (request.type) {
            case OpCode.createContainer:
            case OpCode.create:
            case OpCode.create2:
                CreateRequest create2Request = new CreateRequest();
                pRequest2Txn(request.type, zks.getNextZxid(), request, create2Request, true);
                //... 不重要省略了
    }
}
private final AtomicLong hzxid = new AtomicLong(0);

long getNextZxid() {
    return hzxid.incrementAndGet();
}
```



这个zxid在很多场景都会用到，作为单次客户端发起事务的唯一标识，可作为日志追加记录的文件名（转二进制）；也可以作为zk执行时参考的依据，比如worker执行某个zxid的时候，从事件队列中弹出事件（ChangeRecord）时可以比较zxid的大小，如果小的话说明已经过时了，可以remove掉

```
while (!zks.outstandingChanges.isEmpty()
       && zks.outstandingChanges.peek().zxid <= zxid) {
    ChangeRecord cr = zks.outstandingChanges.remove();
    if (cr.zxid < zxid) {
        LOG.warn("Zxid outstanding " + cr.zxid
                 + " is less than current " + zxid);
    }
    if (zks.outstandingChangesForPath.get(cr.path) == cr) {
        zks.outstandingChangesForPath.remove(cr.path);
    }
}
```



另外存放 节点修改事件的 数据结构是 HashMap，可以保证针对某一个path 的修改操作只会存在最新的一条

```
// this data structure must be accessed under the outstandingChanges lock
final HashMap<String, ChangeRecord> outstandingChangesForPath =
    new HashMap<String, ChangeRecord>();
```



#### 2. 修改 DataNode 的操作，写时复制

有修改事件需要修改父节点时 会 duplicate 一下父节点的最新操作（如果没有则从DataTree中新建一个），写时复制，避免直接修改原事件信息

```
// 获取最近一次针对父节点的操作 如果没有 则新建一个
ChangeRecord parentRecord = getRecordForPath(parentPath);
ChangeRecord nodeRecord = getRecordForPath(path);
// ... 不重要省略了
parentRecord = parentRecord.duplicate(request.getHdr().getZxid());
parentRecord.childCount--;
addChangeRecord(parentRecord);
addChangeRecord(new ChangeRecord(request.getHdr().getZxid(), path, null, -1, null));
```



#### 3. 加锁

典型的例子是需要修改 outstandingChangesForPath 的数据时，都需要进行加锁

```
private ChangeRecord getRecordForPath(String path) throws KeeperException.NoNodeException {
    ChangeRecord lastChange = null;
    synchronized (zks.outstandingChanges) {
        lastChange = zks.outstandingChangesForPath.get(path);
        //...
}

public void processRequest(Request request) {
    synchronized (zks.outstandingChanges) {
        // Need to process local session requests
        // 修改本地内存（DataTree）中的节点信息
        rc = zks.processTxn(request);
        //...
}
```



#### 4. 数据 有快照 + 日志文件保证数据安全

数据可以从日志追加记录中 或者 快照文件 进行恢复，见FileTxnSnapLog.restore() 中的 fastForwardFromEdits方法

```
/**
 * This function will fast forward the server database to have the latest
 * transactions in it.  This is the same as restore, but only reads from
 * the transaction logs and not restores from a snapshot.
 * @param dt the datatree to write transactions to.
 * @param sessions the sessions to be restored.
 * @param listener the playback listener to run on the
 * database transactions.
 * @return the highest zxid restored.
 * @throws IOException
 */
public long fastForwardFromEdits(DataTree dt, Map<Long, Integer> sessions,
                                 PlayBackListener listener) throws IOException {
    TxnIterator itr = txnLog.read(dt.lastProcessedZxid+1);
    long highestZxid = dt.lastProcessedZxid;
    TxnHeader hdr;
    try {
        // 按事务日志的zxid顺序解析所有文件
        while (true) {
            // iterator points to
            // the first valid txn when initialized
            hdr = itr.getHeader();
            if (hdr == null) {
                //empty logs
                return dt.lastProcessedZxid;
            }
            // 更新zxid并处理事务
            if (hdr.getZxid() < highestZxid && highestZxid != 0) {
                LOG.error("{}(highestZxid) > {}(next log) for type {}",
                        highestZxid, hdr.getZxid(), hdr.getType());
            } else {
                highestZxid = hdr.getZxid();
            }
            try {
                processTransaction(hdr,dt,sessions, itr.getTxn());
            } catch(KeeperException.NoNodeException e) {
               throw new IOException("Failed to process transaction type: " +
                     hdr.getType() + " error: " + e.getMessage(), e);
            }
            // 监听器监听事务日志恢复信息
            listener.onTxnLoaded(hdr, itr.getTxn());
            if (!itr.next())
                break;
        }
    } finally {
        if (itr != null) {
            itr.close();
        }
    }
    return highestZxid;
}
```



#### 5. acl 权限安全

每个节点都会有对应的权限，新节点是ALL权限

```
public interface Perms {
    int READ = 1 << 0;

    int WRITE = 1 << 1;

    int CREATE = 1 << 2;

    int DELETE = 1 << 3;

    int ADMIN = 1 << 4;

    int ALL = READ | WRITE | CREATE | DELETE | ADMIN;
}
```



执行相应的事件操作时会检查节点权限

```
// 校验父节点是否有创建节点的权限
checkACL(zks, parentRecord.acl, ZooDefs.Perms.CREATE, request.authInfo);

/**
 * Grant or deny authorization to an operation on a node as a function of:
 *
 * @param zks: not used.
 * @param acl:  set of ACLs for the node
 * @param perm: the permission that the client is requesting
 * @param ids:  the credentials supplied by the client
 */
static void checkACL(ZooKeeperServer zks, List<ACL> acl, int perm,
                     List<Id> ids) throws KeeperException.NoAuthException {
    if (skipACL) {
        return;
    }
    if (acl == null || acl.size() == 0) {
        return;
    }
    for (Id authId : ids) {
        if (authId.getScheme().equals("super")) {
            return;
        }
    }
    for (ACL a : acl) {
        Id id = a.getId();
        if ((a.getPerms() & perm) != 0) {
            if (id.getScheme().equals("world")
                    && id.getId().equals("anyone")) {
                return;
            }
            AuthenticationProvider ap = ProviderRegistry.getProvider(id
                    .getScheme());
            if (ap != null) {
                for (Id authId : ids) {
                    if (authId.getScheme().equals(id.getScheme())
                            && ap.matches(authId.getId(), id.getId())) {
                        return;
                    }
                }
            }
        }
    }
    throw new KeeperException.NoAuthException();
}
```