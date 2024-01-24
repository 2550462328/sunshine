Leaf-snowflake旨在解决基于Leaf-segment下ID递增的问题（不够随机）

![img](http://pcc.huitogo.club/6eb0e3ec25b4fee226ae28de2dd0e274)



Leaf-snowflake方案完全沿用snowflake方案的bit位设计，即是“1+41+10+12”的方式组装ID号。对于workerID的分配，当服务集群数量较小的情况下，完全可以手动配置。Leaf服务规模较大，动手配置成本太高。所以使用Zookeeper持久顺序节点的特性自动对snowflake节点配置wokerID。



Leaf-snowflake是按照下面几个步骤启动的：

1. 启动Leaf-snowflake服务，连接Zookeeper，在leaf_forever父节点下检查自己是否已经注册过（是否有该顺序子节点）。
2. 如果有注册过直接取回自己的workerID（zk顺序节点生成的int类型ID号），启动服务。
3. 如果没有注册过，就在该父节点下面创建一个持久顺序节点，创建成功后取回顺序号当做自己的workerID号，启动服务。

![img](http://pcc.huitogo.club/1483ad8c1b2de6b36fb710029f02eb96)



但是这种方案又会带来新的两个问题



**1）zookeeper强依赖**

怎么弱依赖zookeeper呢？



除了每次会去ZK拿数据以外，也会在本机文件系统上缓存一个workerID文件。当ZooKeeper出现问题，恰好机器出现问题需要重启时，能保证服务能够正常启动。这样做到了对三方组件的弱依赖。一定程度上提高了SLA。



**2）时钟问题**

因为这种方案依赖时间，如果机器的时钟发生了回拨，那么就会有可能生成重复的ID号，需要解决时钟回退的问题。

![img](http://pcc.huitogo.club/2921b856867f865c90231fc4f10870b0)



基于上述方案做了对时间求平均值和准确性判断处理，现在Leaf-snowflake的启动如下

1. 服务启动时首先检查自己是否写过ZooKeeper leaf_forever节点
2. 若写过，则用自身系统时间与leaf_forever/${self}节点记录时间做比较，若小于leaf_forever/${self}时间则认为机器时间发生了大步长回拨，服务启动失败并报警。
3. 若未写过，证明是新服务节点，直接创建持久节点leaf_forever/${self}并写入自身系统时间，接下来综合对比其余Leaf节点的系统时间来判断自身系统时间是否准确，具体做法是取leaf_temporary下的所有临时节点(所有运行中的Leaf-snowflake节点)的服务IP：Port，然后通过RPC请求得到所有节点的系统时间，计算sum(time)/nodeSize。
4. 若abs( 系统时间-sum(time)/nodeSize ) < 阈值，认为当前系统时间准确，正常启动服务，同时写临时节点leaf_temporary/${self} 维持租约。
5. 否则认为本机系统时间发生大步长偏移，启动失败并报警。
6. 每隔一段时间(3s)上报自身系统时间写入leaf_forever/${self}。



由于强依赖时钟，对时间的要求比较敏感，在机器工作时NTP同步也会造成秒级别的回退，建议可以直接关闭NTP同步。要么在时钟回拨的时候直接不提供服务直接返回ERROR_CODE，等时钟追上即可。或者做一层重试，然后上报报警系统，更或者是发现有时钟回拨之后自动摘除本身节点并报警，如下：

```
1. //发生了回拨，此刻时间小于上次发号时间 

2. if (timestamp < lastTimestamp) { 

3.         

4.      long offset = lastTimestamp - timestamp; 

5.      if (offset <= 5) { 

6.        try { 

7.         //时间偏差大小小于5ms，则等待两倍时间 

8.          wait(offset << 1);//wait 

9.          timestamp = timeGen(); 

10.          if (timestamp < lastTimestamp) { 

11.            //还是小于，抛异常并上报 

12.            throwClockBackwardsEx(timestamp); 

13.           }   

14.        } catch (InterruptedException e) {  

15.          throw e; 

16.        } 

17.      } else { 

18.        //throw 

19.        throwClockBackwardsEx(timestamp); 

20.      } 

21.    } 

22. //分配ID  
```



Leaf在美团点评公司内部服务包含金融、支付交易、餐饮、外卖、酒店旅游、猫眼电影等众多业务线。目前Leaf的性能在4C8G的机器上QPS能压测到近5w/s，TP999 1ms，已经能够满足大部分的业务的需求。每天提供亿数量级的调用量，作为公司内部公共的基础技术设施，必须保证高SLA和高性能的服务，我们目前还仅仅达到了及格线，还有很多提高的空间。