为更好演示Leader Failover过程，本例中共使用5个ZooKeeper服务器。A作为Leader，共收到P1、P2、P3三条消息，并且Commit了1和2，且总体顺序为P1、P2、C1、P3、C2。根据顺序性原则，其它Follower收到的消息的顺序肯定与之相同。其中B与A完全同步，C收到P1、P2、C1，D收到P1、P2，E收到P1，如下图所示。

![img](http://pcc.huitogo.club/9bba64b399d227b156686164272ab7c3)



这里要注意：

1. 由于A没有C3，意味着收到P3的服务器的总个数不会超过一半，也即包含A在内最多只有两台服务器收到P3。在这里A和B收到P3，其它服务器均未收到P3
2. 由于A已写入C1、C2，说明它已经Commit了P1、P2，因此整个集群有超过一半的服务器，即最少三个服务器收到P1、P2。在这里所有服务器都收到了P1，除E外其它服务器也都收到了P2



#### 1. 选出新Leader

旧Leader也即A宕机后，其它服务器根据上述FastLeaderElection算法选出B作为新的Leader。C、D和E成为Follower且以B为Leader后，会主动将自己最大的zxid发送给B，B会将Follower的zxid与自身zxid间的所有被Commit过的消息同步给Follower，如下图所示。

![img](http://pcc.huitogo.club/2c38b2deda8924ca8c91c7318fc48e00)



在上图中：

1. P1和P2都被A Commit，因此B会通过同步保证P1、P2、C1与C2都存在于C、D和E中
2. P3由于未被A Commit，同时幸存的所有服务器中P3未存在于大多数据服务器中，因此它不会被同步到其它Follower



#### 2. 通知Follower可对外服务

同步完数据后，B会向D、C和E发送NEWLEADER命令并等待大多数服务器的ACK（下图中D和E已返回ACK，加上B自身，已经占集群的大多数），然后向所有服务器广播UPTODATE命令。收到该命令后的服务器即可对外提供服务。

![img](http://pcc.huitogo.club/6b66427d60c1da6808fa5b88f5c335bb)



需要注意的是**未Commit过的消息对客户端不可见**

在上例中，P3未被A Commit过，同时因为没有过半的服务器收到P3，因此B也未Commit P3（如果有过半服务器收到P3，即使A未Commit P3，B会主动Commit P3，即C3），所以它不会将P3广播出去。



具体做法是，B在成为Leader后，先判断自身未Commit的消息（本例中即P3）是否存在于大多数服务器中从而决定是否要将其Commit。然后B可得出自身所包含的被Commit过的消息中的最小zxid（记为min_zxid）与最大zxid（记为max_zxid）。C、D和E向B发送自身Commit过的最大消息zxid（记为max_zxid）以及未被Commit过的所有消息（记为zxid_set）。



B根据这些信息作出如下操作：

1. 如果Follower的max_zxid与Leader的max_zxid相等，说明该Follower与Leader完全同步，无须同步任何数据
2. 如果Follower的max_zxid在Leader的(min_zxid，max_zxid)范围内，Leader会通过TRUNC命令通知Follower将其zxid_set中大于Follower的max_zxid（如果有）的所有消息全部删除



上述操作保证了未被Commit过的消息不会被Commit从而对外不可见。



上述例子中Follower上并不存在未被Commit的消息。但可考虑这种情况，如果将上述例子中的服务器数量从五增加到七，服务器F包含P1、P2、C1、P3，服务器G包含P1、P2。此时服务器F、A和B都包含P3，但是因为票数未过半，因此B作为Leader不会Commit P3，而会通过TRUNC命令通知F删除P3。如下图所示。

![img](http://pcc.huitogo.club/f346a12c831fcf062b9972d462ffadad)