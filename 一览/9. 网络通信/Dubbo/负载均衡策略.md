在集群负载均衡时，Dubbo 提供了多种均衡策略，**默认为 random 随机调用**。



#### 1. Random LoadBalance(默认，基于权重的随机负载均衡机制)

**随机，按权重设置随机概率。**

在一个截面上碰撞的概率高，但调用量越大分布越均匀，而且按概率使用权重后也比较均匀，有利于动态调整提供者权重。

![img](http://pcc.huitogo.club/3b048bd4f45650a5fd5b34d60875f457)



#### 2. RoundRobin LoadBalance(不推荐，基于权重的轮询负载均衡机制)

**轮循，按公约后的权重设置轮循比率。**

存在慢的提供者累积请求的问题，比如：第二台机器很慢，但没挂，当请求调到第二台时就卡在那，久而久之，所有请求都卡在调到第二台上。

![img](http://pcc.huitogo.club/698fc32922e723eb66e0e74686d73541)



#### 3. LeastActive LoadBalance

最少活跃调用数，相同活跃数的随机，活跃数指调用前后计数差。

使慢的提供者收到更少请求，因为越慢的提供者的调用前后计数差会越大。



#### 4. ConsistentHash LoadBalance

一致性 Hash，相同参数的请求总是发到同一提供者。(如果你需要的不是随机负载均衡，是要一类请求都到一个节点，那就走这个一致性hash策略。)



当某一台提供者挂时，原本发往该提供者的请求，基于虚拟节点，平摊到其它提供者，不会引起剧烈变动。

缺省只对第一个参数 Hash，如果要修改，请配置 <dubbo:parameter key="hash.arguments" value="0,1" />

缺省用 160 份虚拟节点，如果要修改，请配置<dubbo:parameter key="hash.nodes" value="320" />



**如何配置负载均衡策略？**

1）服务级别

```
1. <dubbo:service interface="..." loadbalance="roundrobin" /> 
```



2）服务方法级别

```
1. <dubbo:service interface="..."> 

2.   <dubbo:method name="..." loadbalance="roundrobin"/> 

3. </dubbo:service> 
```



3）基于注解的配置

```
1. @Reference(loadbalance = "roundrobin") 

2. HelloService helloService; 
```