**什么是ZAB协议？**

为了保证写操作的一致性与可用性，ZooKeeper专门设计了一种名为原子广播（ZAB）的支持崩溃恢复的一致性协议。基于该协议，ZooKeeper实现了一种主从模式的系统架构来保持集群中各个副本之间的数据一致性。

根据ZAB协议，所有的写操作都必须通过Leader完成，Leader写入本地日志后再复制到所有的Follower节点。

一旦Leader节点无法工作，ZAB协议能够自动从Follower节点中重新选出一个合适的替代者，即新的Leader，该过程即为领导选举。该领导选举过程，是ZAB协议中最为重要和复杂的过程。



**ZAB协议的选举算法**

可通过electionAlg配置项设置ZooKeeper用于领导选举的算法。

到3.4.10版本为止，可选项有：

- 0 基于UDP的LeaderElection
- 1 基于UDP的FastLeaderElection
- 2 基于UDP和认证的FastLeaderElection
- **3 基于TCP的FastLeaderElection**



在3.4.10版本中，默认值为3，也即基于TCP的FastLeaderElection。另外三种算法已经被弃用，并且有计划在之后的版本中将它们彻底删除而不再支持。

所以我们主要讨论**FastLeaderElection算法的原理**

#### 1. myid

每个ZooKeeper服务器，都需要在数据文件夹下创建一个名为myid的文件，该文件包含整个ZooKeeper集群唯一的ID（整数）。例如，某ZooKeeper集群包含三台服务器，hostname分别为zoo1、zoo2和zoo3，其myid分别为1、2和3，则在配置文件中其ID与hostname必须一一对应，如下所示。在该配置文件中，server.后面的数据即为myid。



#### 2. zxid

类似于RDBMS中的事务ID，用于标识一次更新操作的Proposal ID。为了保证顺序性，该zkid必须单调递增。因此ZooKeeper使用一个64位的数来表示，高32位是Leader的epoch，从1开始，每次选出新的Leader，epoch加一。低32位为该epoch内的序号，每次epoch变化，都将低32位的序号重置。这样保证了zkid的全局递增性。



#### 3. 服务器状态

- LOOKING 不确定Leader状态。该状态下的服务器认为当前集群中没有Leader，会发起Leader选举。
- FOLLOWING 跟随者状态。表明当前服务器角色是Follower，并且它知道Leader是谁。
- LEADING 领导者状态。表明当前服务器角色是Leader，它会维护与Follower间的心跳。
- OBSERVING 观察者状态。表明当前服务器角色是Observer，与Folower唯一的不同在于不参与选举，也不参与集群写操作时的投票。



#### 4. 选票数据结构

每个服务器在进行领导选举时，会发送如下关键信息：

- logicClock 每个服务器会维护一个自增的整数，名为logicClock，它表示这是该服务器发起的第多少轮投票
- state 当前服务器的状态
- self_id 当前服务器的myid
- self_zxid 当前服务器上所保存的数据的最大zxid
- vote_id 被推举的服务器的myid
- vote_zxid 被推举的服务器上所保存的数据的最大zxid



#### 5. 投票流程

**1）自增选举轮次**

ZooKeeper规定所有有效的投票都必须在同一轮次中。每个服务器在开始新一轮投票时，会先对自己维护的logicClock进行自增操作。



**2）初始化选票**

每个服务器在广播自己的选票前，会将自己的投票箱清空。该投票箱记录了所收到的选票。例：服务器2投票给服务器3，服务器3投票给服务器1，则服务器1的投票箱为(2, 3), (3, 1), (1, 1)。票箱中只会记录每一投票者的最后一票，如投票者更新自己的选票，则其它服务器收到该新选票后会在自己票箱中更新该服务器的选票。



**3）发送初始化选票**

每个服务器最开始都是通过广播把票投给自己。



**4）接收外部投票**

服务器会尝试从其它服务器获取投票，并记入自己的投票箱内。如果无法获取任何外部投票，则会确认自己是否与集群中其它服务器保持着有效连接。如果是，则再次发送自己的投票；如果否，则马上与之建立连接。



**5）判断选举轮次**

收到外部投票后，首先会根据投票信息中所包含的logicClock来进行不同处理：

1. 外部投票的logicClock大于自己的logicClock。说明该服务器的选举轮次落后于其它服务器的选举轮次，立即清空自己的投票箱并将自己的logicClock更新为收到的logicClock，然后再对比自己之前的投票与收到的投票以确定是否需要变更自己的投票，最终再次将自己的投票广播出去。
2. 外部投票的logicClock小于自己的logicClock。当前服务器直接忽略该投票，继续处理下一个投票。
3. 外部投票的logickClock与自己的相等。当时进行选票PK。



**6）选票PK**

选票PK是基于(self_id, self_zxid)与(vote_id, vote_zxid)的对比：

1. 外部投票的logicClock大于自己的logicClock，则将自己的logicClock及自己的选票的logicClock变更为收到的logicClock
2. 若logicClock一致，则对比二者的vote_zxid，若外部投票的vote_zxid比较大，则将自己的票中的vote_zxid与vote_myid更新为收到的票中的vote_zxid与vote_myid并广播出去，另外将收到的票及自己更新后的票放入自己的票箱。如果票箱内已存在(self_myid, self_zxid)相同的选票，则直接覆盖
3. 若二者vote_zxid一致，则比较二者的vote_myid，若外部投票的vote_myid比较大，则将自己的票中的vote_myid更新为收到的票中的vote_myid并广播出去，另外将收到的票及自己更新后的票放入自己的票箱



**7）统计选票**

如果已经确定有过半服务器认可了自己的投票（可能是更新后的投票），则终止投票。否则继续接收其它服务器的投票。



**8）更新服务器状态**

投票终止后，服务器开始更新自身状态。若过半的票投给了自己，则将自己的服务器状态更新为LEADING，否则将自己的状态更新为FOLLOWING。