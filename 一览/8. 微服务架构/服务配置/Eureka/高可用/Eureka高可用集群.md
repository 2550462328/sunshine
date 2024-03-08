![img](https://img-blog.csdnimg.cn/ab97522f59c6477bb595c8cf6554037f.png)

上面的图有三台 Eureka Server 组成的集群，每一台 Eureka Server服务在不同的地方。这样三台 Eureka Server 就组建成了一个高可用集群服务，只要三个服务有一个能一直正常运行，就不会影响整个架构的稳定性。