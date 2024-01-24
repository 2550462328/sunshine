先总结一下，数据 分别存在内存 和 磁盘中，通常 对节点的 CRUD发生在内存中（不难理解，快嘛），以DataTree（TreeNode）的数据结构进行存储； 持久化入磁盘的以l snapshot 文件的方式进行存储，可用于 初始化数据、数据恢复等后台耗时操作；还有一种内存数据用于主从节点的数据同步，基于 log 的数据结构；



对数据的操作都在ZkDatabase中

```
public class ZKDatabase {

    private static final Logger LOG = LoggerFactory.getLogger(ZKDatabase.class);

    /**
     * make sure on a clear you take care of all these members.
     */
    protected DataTree dataTree;
    protected ConcurrentHashMap<Long, Integer> sessionsWithTimeouts;
    //FileTxnSnapLog是管理日志文件和持久化文件的对象
    protected FileTxnSnapLog snapLog;
    protected long minCommittedLog, maxCommittedLog;

    /**
     * Default value is to use snapshot if txnlog size exceeds 1/3 the size of snapshot
     */
    public static final String SNAPSHOT_SIZE_FACTOR = "zookeeper.snapshotSizeFactor";
    public static final double DEFAULT_SNAPSHOT_SIZE_FACTOR = 0.33;
    private double snapshotSizeFactor;

    public static final int commitLogCount = 500;
    protected static int commitLogBuffer = 700;
    protected LinkedList<Proposal> committedLog = new LinkedList<Proposal>();
    protected ReentrantReadWriteLock logLock = new ReentrantReadWriteLock();
    volatile private boolean initialized = false;
    ...    
}
```



数据节点的类型有以下几种

```
/**
 * The znode will not be automatically deleted upon client's disconnect.
 */
PERSISTENT (0, false, false, false, false),
/**
* The znode will not be automatically deleted upon client's disconnect,
* and its name will be appended with a monotonically increasing number.
*/
PERSISTENT_SEQUENTIAL (2, false, true, false, false),
/**
 * The znode will be deleted upon the client's disconnect.
 */
EPHEMERAL (1, true, false, false, false),
/**
 * The znode will be deleted upon the client's disconnect, and its name
 * will be appended with a monotonically increasing number.
 */
EPHEMERAL_SEQUENTIAL (3, true, true, false, false),
/**
 * The znode will be a container node. Container
 * nodes are special purpose nodes useful for recipes such as leader, lock,
 * etc. When the last child of a container is deleted, the container becomes
 * a candidate to be deleted by the server at some point in the future.
 * Given this property, you should be prepared to get
 */
CONTAINER (4, false, false, true, false),
/**
 * The znode will not be automatically deleted upon client's disconnect.
 * However if the znode has not been modified within the given TTL, it
 * will be deleted once it has no children.
 */
PERSISTENT_WITH_TTL(5, false, false, false, true),
/**
 * The znode will not be automatically deleted upon client's disconnect,
 * and its name will be appended with a monotonically increasing number.
 * However if the znode has not been modified within the given TTL, it
 * will be deleted once it has no children.
 */
PERSISTENT_SEQUENTIAL_WITH_TTL(6, false, true, false, true);
```



还是以创建节点为例，先看一下内存中怎么 插入节点的，内存的核心操作类是DataTree，看一下DataTree.createNode()

```
/**
 * Add a new node to the DataTree.
 * @param path
 *             Path for the new node.
 * @param data
 *            Data to store in the node.
 * @param acl
 *            Node acls
 * @param ephemeralOwner
 *            the session id that owns this node. -1 indicates this is not
 *            an ephemeral node.
 * @param zxid
 *            Transaction ID
 * @param time
 * @param outputStat
 *             A Stat object to store Stat output results into.
 * @throws NodeExistsException
 * @throws NoNodeException
 * @throws KeeperException
 */
public void createNode(final String path, byte data[], List<ACL> acl,
        long ephemeralOwner, int parentCVersion, long zxid, long time, Stat outputStat)
        throws KeeperException.NoNodeException,
        KeeperException.NodeExistsException {
    int lastSlash = path.lastIndexOf('/');
    String parentName = path.substring(0, lastSlash);
    String childName = path.substring(lastSlash + 1);
    StatPersisted stat = new StatPersisted();
    stat.setCtime(time);
    stat.setMtime(time);
    stat.setCzxid(zxid);
    stat.setMzxid(zxid);
    stat.setPzxid(zxid);
    stat.setVersion(0);
    stat.setAversion(0);
    stat.setEphemeralOwner(ephemeralOwner);
    DataNode parent = nodes.get(parentName);
    if (parent == null) {
        throw new KeeperException.NoNodeException();
    }
    synchronized (parent) {
        Set<String> children = parent.getChildren();
        if (children.contains(childName)) {
            throw new KeeperException.NodeExistsException();
        }

        if (parentCVersion == -1) {
            parentCVersion = parent.stat.getCversion();
            parentCVersion++;
        }
        // 1. 修改父节点
        parent.stat.setCversion(parentCVersion);
        parent.stat.setPzxid(zxid);
        // 根据acl 转换成引用数量值
        // acl是节点的权限信息
        Long longval = aclCache.convertAcls(acl);
        DataNode child = new DataNode(data, longval, stat);
        parent.addChild(childName);
        // 2. 添加新增节点
        nodes.put(path, child);
        // 3. 将新增节点分类
        EphemeralType ephemeralType = EphemeralType.get(ephemeralOwner);
        if (ephemeralType == EphemeralType.CONTAINER) {
            containers.add(path);
        } else if (ephemeralType == EphemeralType.TTL) {
            ttls.add(path);
        } else if (ephemeralOwner != 0) {
            HashSet<String> list = ephemerals.get(ephemeralOwner);
            if (list == null) {
                list = new HashSet<String>();
                ephemerals.put(ephemeralOwner, list);
            }
            synchronized (list) {
                list.add(path);
            }
        }
        // 4. 复制出新增节点 状态信息 （用于返回）
        if (outputStat != null) {
           child.copyStat(outputStat);
        }
    }
    // now check if its one of the zookeeper node child
    // 5. 判断是否到系统节点下（比如手动添加引用信息）
    if (parentName.startsWith(quotaZookeeper)) {
        // now check if its the limit node
        if (Quotas.limitNode.equals(childName)) {
            // this is the limit node
            // get the parent and add it to the trie
            pTrie.addPath(parentName.substring(quotaZookeeper.length()));
        }
        if (Quotas.statNode.equals(childName)) {
            updateQuotaForPath(parentName
                    .substring(quotaZookeeper.length()));
        }
    }
    // also check to update the quotas for this node
    // 6. 修改节点的引用数
    String lastPrefix = getMaxPrefixWithQuota(path);
    if(lastPrefix != null) {
        // ok we have some match and need to update
        updateCount(lastPrefix, 1);
        updateBytes(lastPrefix, data == null ? 0 : data.length);
    }
    // 触发节点 watch事件
    dataWatches.triggerWatch(path, Event.EventType.NodeCreated);
    childWatches.triggerWatch(parentName.equals("") ? "/" : parentName,
            Event.EventType.NodeChildrenChanged);
}
```



看下第二种写入内存中的 log 格式，用于主从之间的数据同步

```
/**
 * maintains a list of last <i>committedLog</i>
 *  or so committed requests. This is used for
 * fast follower synchronization.
 * @param request committed request
 */
public void addCommittedProposal(Request request) {
    // 读写锁
    WriteLock wl = logLock.writeLock();
    try {
        wl.lock();
        // 现在最大提交日志记录大小默认为500
        if (committedLog.size() > commitLogCount) {
            committedLog.removeFirst();
            // 挤出一个后取最小的
            minCommittedLog = committedLog.getFirst().packet.getZxid();
        }
        if (committedLog.isEmpty()) {
            minCommittedLog = request.zxid;
            maxCommittedLog = request.zxid;
        }

        byte[] data = SerializeUtils.serializeRequest(request);
        // 定制日志包信息
        QuorumPacket pp = new QuorumPacket(Leader.PROPOSAL, request.zxid, data, null);
        Proposal p = new Proposal();
        p.packet = pp;
        p.request = request;
        // 新增一条提议 = 定制日志包 + request信息
        committedLog.add(p);
        // 修改最大提交日志偏移量
        maxCommittedLog = p.packet.getZxid();
    } finally {
        wl.unlock();
    }
}
```



committedLog 默认最多可以存储500条日志记录，超出时 则删除最老一条数据



在看磁盘操作时，我们先回过头看一下 请求数据处理的第二条处理链 SyncRequestProcessor.processRequest()

```
public void processRequest(Request request) {
    queuedRequests.add(request);
}
```



这里同样使用一条队列实现异步操作，实际执行代码在SyncRequestProcessor.run()中

```
public void run() {
    try {
        // 生成快照数量
        int logCount = 0;

        // we do this in an attempt to ensure that not all of the servers
        // in the ensemble take a snapshot at the same time
        int randRoll = r.nextInt(snapCount/2);
        while (true) {
            Request si = null;
            if (toFlush.isEmpty()) {
                si = queuedRequests.take();
            } else {
                si = queuedRequests.poll();
                if (si == null) {
                    flush(toFlush);
                    continue;
                }
            }
            if (si == requestOfDeath) {
                break;
            }
            if (si != null) {
                // track the number of records written to the log
                // 追加日志记录 .log.21212
                if (zks.getZKDatabase().append(si)) {
                    logCount++;
                    // 生成快照数量过多过快时，进行rollback
                    if (logCount > (snapCount / 2 + randRoll)) {
                        randRoll = r.nextInt(snapCount/2);
                        // roll the log
                        zks.getZKDatabase().rollLog();
                        // take a snapshot
                        // 生成快照的快照线程
                        if (snapInProcess != null && snapInProcess.isAlive()) {
                            LOG.warn("Too busy to snap, skipping");
                        } else {
                            snapInProcess = new ZooKeeperThread("Snapshot Thread") {
                                    public void run() {
                                        try {
                                            zks.takeSnapshot();
                                        } catch(Exception e) {
                                            LOG.warn("Unexpected exception", e);
                                        }
                                    }
                                };
                            snapInProcess.start();
                        }
                        logCount = 0;
                    }
                } else if (toFlush.isEmpty()) {
                    // optimization for read heavy workloads
                    // iff this is a read, and there are no pending
                    // flushes (writes), then just pass this to the next
                    // processor
                    if (nextProcessor != null) {
                        nextProcessor.processRequest(si);
                        if (nextProcessor instanceof Flushable) {
                            ((Flushable)nextProcessor).flush();
                        }
                    }
                    continue;
                }
                toFlush.add(si);
                // 追加记录达到1000条进行落盘（刷入磁盘）
                if (toFlush.size() > 1000) {
                    // 落盘 即 
                    flush(toFlush);
                }
            }
        }
    } catch (Throwable t) {
        handleException(this.getName(), t);
    } finally{
        running = false;
    }
    LOG.info("SyncRequestProcessor exited!");
}
```



实际存储磁盘有两种形式，一种是追加日志记录 .log ；一种是以快照 snapshot 的方式 存储，相关操作在 FileTxnSnapLog 中



追加日志记录 FileTxnSnapLog.append()

```
public synchronized boolean append(TxnHeader hdr, Record txn)
    throws IOException
{
    if (hdr == null) {
        return false;
    }
    if (hdr.getZxid() <= lastZxidSeen) {
        LOG.warn("Current zxid " + hdr.getZxid()
                + " is <= " + lastZxidSeen + " for "
                + hdr.getType());
    } else {
        lastZxidSeen = hdr.getZxid();
    }
    if (logStream==null) {
       if(LOG.isInfoEnabled()){
            LOG.info("Creating new log file: " + Util.makeLogName(hdr.getZxid()));
       }

       logFileWrite = new File(logDir, Util.makeLogName(hdr.getZxid()));
       fos = new FileOutputStream(logFileWrite);
       logStream=new BufferedOutputStream(fos);
       oa = BinaryOutputArchive.getArchive(logStream);
       FileHeader fhdr = new FileHeader(TXNLOG_MAGIC,VERSION, dbId);
       fhdr.serialize(oa, "fileheader");
       // Make sure that the magic number is written before padding.
       logStream.flush();
       filePadding.setCurrentSize(fos.getChannel().position());
       streamsToFlush.add(fos);
    }
    filePadding.padFile(fos.getChannel());
    byte[] buf = Util.marshallTxnEntry(hdr, txn);
    if (buf == null || buf.length == 0) {
        throw new IOException("Faulty serialization for header " +
                "and txn");
    }
    Checksum crc = makeChecksumAlgorithm();
    crc.update(buf, 0, buf.length);
    oa.writeLong(crc.getValue(), "txnEntryCRC");
    Util.writeTxnBytes(oa, buf);

    return true;
}
```



追加记录的 输出流会预先保存在streamsToFlush中，等待toFlush的队列数量达到1000条即可刷入磁盘，即FileTxnSnapLog.commit()

```
/**
 * commit the logs. make sure that everything hits the
 * disk
 */
public synchronized void commit() throws IOException {
    if (logStream != null) {
        logStream.flush();
    }
    for (FileOutputStream log : streamsToFlush) {
        log.flush();
        if (forceSync) {
            long startSyncNS = System.nanoTime();

            FileChannel channel = log.getChannel();
            channel.force(false);

            syncElapsedMS = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startSyncNS);
            if (syncElapsedMS > fsyncWarningThresholdMS) {
                if(serverStats != null) {
                    serverStats.incrementFsyncThresholdExceedCount();
                }
                LOG.warn("fsync-ing the write ahead log in "
                        + Thread.currentThread().getName()
                        + " took " + syncElapsedMS
                        + "ms which will adversely effect operation latency. "
                        + "File size is " + channel.size() + " bytes. "
                        + "See the ZooKeeper troubleshooting guide");
            }
        }
    }
    while (streamsToFlush.size() > 1) {
        streamsToFlush.removeFirst().close();
    }
}
```



再看一下快照文件的生成方式，快照是由snapInProcess线程异步执行的，有新增追加记录的时候进行触发，生成快照的代码如下

```
public void takeSnapshot(){
    try {
        txnLogFactory.save(zkDb.getDataTree(), zkDb.getSessionWithTimeOuts());
    } catch (IOException e) {
        LOG.error("Severe unrecoverable error, exiting", e);
        // This is a severe error that we cannot recover from,
        // so we need to exit
        System.exit(10);
    }
}

/**
 * save the datatree and the sessions into a snapshot
 * @param dataTree the datatree to be serialized onto disk
 * @param sessionsWithTimeouts the session timeouts to be
 * serialized onto disk
 * @throws IOException
 */
public void save(DataTree dataTree,
        ConcurrentHashMap<Long, Integer> sessionsWithTimeouts)
    throws IOException {
    long lastZxid = dataTree.lastProcessedZxid;
    File snapshotFile = new File(snapDir, Util.makeSnapshotName(lastZxid));
    LOG.info("Snapshotting: 0x{} to {}", Long.toHexString(lastZxid),
            snapshotFile);
    snapLog.serialize(dataTree, sessionsWithTimeouts, snapshotFile);
}
```



快照文件生成过快（不一定可以全部同步到子节点）时会触发一个rollback操作，也是调用FileTxnSnapLog.rollLog()

```
/**
 * rollover the current log file to a new one.
 * @throws IOException
 */
public synchronized void rollLog() throws IOException {
    if (logStream != null) {
        this.logStream.flush();
        this.logStream = null;
        oa = null;
    }
}
```



最后顺便看一下 磁盘里的log文件怎么恢复成 内存中的 DataTree结构，这里调用了FileTxnSnapLog.restore()方法

```
/**
 * this function restores the server
 * database after reading from the
 * snapshots and transaction logs
 * @param dt the datatree to be restored
 * @param sessions the sessions to be restored
 * @param listener the playback listener to run on the
 * database restoration
 * @return the highest zxid restored
 * @throws IOException
 */
public long restore(DataTree dt, Map<Long, Integer> sessions,
                    PlayBackListener listener) throws IOException {
    // 1. 选取100条快照文件，找出最有效最新的一条，将文件的后缀名（二进制）转十进制即最近的一次事务id
    long deserializeResult = snapLog.deserialize(dt, sessions);
    FileTxnLog txnLog = new FileTxnLog(dataDir);

    // 快速的 根据日志记录 文件（列表） 获取
    RestoreFinalizer finalizer = () -> {
        long highestZxid = fastForwardFromEdits(dt, sessions, listener);
        return highestZxid;
    };
    if (-1L == deserializeResult) {
        /* this means that we couldn't find any snapshot, so we need to
         * initialize an empty database (reported in ZOOKEEPER-2325) */
        // 2. 从日志追加记录里面获取最近的一次事务id
        if (txnLog.getLastLoggedZxid() != -1) {
            // ZOOKEEPER-3056: provides an escape hatch for users upgrading
            // from old versions of zookeeper (3.4.x, pre 3.5.3).
            if (!trustEmptySnapshot) {
                throw new IOException(EMPTY_SNAPSHOT_WARNING + "Something is broken!");
            } else {
                LOG.warn("{}This should only be allowed during upgrading.", EMPTY_SNAPSHOT_WARNING);
                return finalizer.run();
            }
        }
        // 3. 对DataTree 进行一次快照
        save(dt, (ConcurrentHashMap<Long, Integer>)sessions);
        /* return a zxid of zero, since we the database is empty */
        return 0;
    }

    return finalizer.run();
}
```