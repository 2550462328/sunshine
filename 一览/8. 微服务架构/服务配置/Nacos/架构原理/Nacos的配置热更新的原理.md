#### 1. Nacos配置热更新的基本原理

Nacos配置热更新的基本原理是通过客户端和Nacos Server之间的长连接实现的。当配置发生变化时，Nacos Server会向订阅了该配置的客户端发送通知，客户端接收到通知后会重新获取最新的配置并更新。 具体的步骤如下：

1. Nacos Server：Nacos Server 是一个集中式的配置中心，负责管理所有的配置信息。
2. Nacos Client：Nacos Client 是应用程序中的一个库，负责与 Nacos Server 进行通信。
3. 配置注册：应用程序启动时，会使用 Nacos Client 将自己的配置注册到 Nacos Server 上，以便 Nacos Server 知道该应用程序的存在。
4. 配置监听：应用程序可以通过 Nacos Client 注册一个监听器，用于监听配置的变化。当配置发生变化时，Nacos Server 会通知 Nacos Client，然后 Nacos Client 会触发监听器的回调方法。
5. 配置更新：当 Nacos Client 接收到配置变化的通知后，会重新从 Nacos Server 获取最新的配置，并更新应用程序的配置。
6. 配置缓存：为了提高性能，Nacos Client 会缓存配置信息。当配置发生变化时，Nacos Client 会先更新缓存中的配置，然后再触发监听器的回调方法。 

通过上述步骤，Nacos 实现了配置的热更新。应用程序只需要注册监听器，当配置发生变化时，就能够自动获取最新的配置并更新。这样就可以实现应用程序在运行时动态调整配置，而不需要重启应用程序。



#### 2.  配置热更新的实现细节

- **长连接**

Nacos通过使用长连接的方式实现了与客户端的实时通信。通过长连接，Nacos Server能够主动向客户端发送配置变更的通知。

- **订阅与通知**

Nacos提供了订阅和通知的机制，客户端可以通过订阅指定的配置来实现热更新。当配置发生变化时，Nacos Server会向订阅了该配置的客户端发送通知。

- **配置的存储和管理**

Nacos将配置信息存储在自身的数据库中，并提供了管理接口供用户进行配置的创建、修改和删除等操作。当配置发生变化时，Nacos Server会将最新的配置信息发送给订阅者。

- **配置的获取和更新**

客户端可以通过向Nacos Server发送请求获取最新的配置信息。当配置发生变化时，客户端会收到Nacos Server的通知，并重新向Nacos Server请求最新的配置信息。



#### 3. SpringBoot中怎么实现热更新？

##### 3.1 实现热更新的两种方式

- **@RefreshScope**

通过 **@Value** 注入后，结合注解 **@RefreshScope** 刷新配置。

过程是在要调用这些变化的配置的类中，通过注解 **@Value** 找到在 **nacos** 中配置的属性，然后在调用这些属性的类上加上注解 **@RefreshScope** 实现配置的自动更新。



- **@ConfigurationProperties + @EnableConfigurationProperties**

重新定义一个方法或者类(该注解使用于类和方法)，通过注解 **@ConfigurationProperties** 的属性 **prefix** 绑定配置文件中的配置，相当于捕捉到外部的配置信息，@EnableConfigurationProperties 注解实现把配置交给 **spring** 管理，实现配置文件的自动刷新。



##### 3.2 @RefreshScope做了什么

@RefreshScope主要就是基于@Scope注解的作用域代理的基础上进行扩展实现的，加了@RefreshScope注解的类，在被Bean工厂创建后会加入自己的refresh scope 这个Bean缓存中，后续会优先从Bean缓存中获取，当配置中心发生了变更，会把变更的配置更新到spring容器的Environment中，并且同时bean缓存就会被清空，从而就会从bean工厂中创建bean实例了，而这次创建bean实例的时候就会继续经历这个bean的生命周期，使得@Value属性值能够从Environment中获取到最新的属性值，这样整个过程就达到了动态刷新配置的效果。



所以截止目前我们可以了解到下述大致流程：

![img](https://pcc.huitogo.club/z0/1676278620132-6b06b93b-a48f-4b97-80b1-ebf437025221.jpeg)



##### 3.3 @RefreshScope怎么做到的？

基于上面我们知道，@Scope基于缓存失效，实现配置的热更新，我们继续看看它是如何做到的：

![img](https://pcc.huitogo.club/z0/1676278632106-281bcaaf-8b97-4344-bd34-4ea0a57eccf5.png)

- nacosClient和nacosServer通信，nacosClient订阅某个配置的事件，nacosServer在配置更新后会通知nacosClient
- nacosClient会更新Enviroment并删除refreshScope的bean的缓存，具体是GenericScope
- nacosClient会publish  RefreshEvent事件由业务方自行实现
- @RefreshScope注解的Bean会被注册成FactoryBean，getObject获取代理，代理根据scope从GenericScope缓存中取值，如果缓存没有则doCreateBean
- doCreateBean会根据最新的Enviroment创建Bean，则保证@Value值是最新的



总结：对于@RefreshScope注解实现配置热更新的流程，实际是借助于缓存失效+Spring重新创建配置Bean解决



