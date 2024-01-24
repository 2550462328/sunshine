NacosNamingService源码分析（看源码的思路）

```
NacosNamingService

服务端 和 监听者（其他服务端、客户端或者第三方监听服务）

Publish | Subscriber    发布订阅
		NotifyCenter    服务注册 | 注销 | 订阅 | 取消订阅
				EventPublisher
						DefaultSharePublisher  多个事件类型 共享一个事件队列   shareBufferSize
						DefaultPublisher    每个事件类型对应一个 事件队列 ringBufferSize

				Subscriber
						SmartSubscriber
								ClientServiceIndexesManager  客户端事件处理  通常会回调一个 监听客户端同步事件
								DistroClientDataProcessor  nacos 集群同步信息 
										DistroProtocol  真正执行同步
								NamingMetadataManager
								NamingSubscriberServiceV2Impl  监听的客户端同步信息 处理
										NacosDelayTaskExecuteEngine  merge 多个客户端监听事件 并执行
												NacosTaskProcessor  process  真正执行同步
```



客户端 和 服务端通信

```
NamingClientProxy  实际执行
		NamingHttpClientProxy 长连接
				BeatReactor  心跳机制
				
		NamingGrpcClientProxy 短连接
				GrpcRequestAcceptor
				RequestHandler
						DistroDataRequestHandler
						HealthCheckRequestHandler
						InstanceRequestHandler   注册节点
						ServerLoaderInfoRequestHandler
						ServerReloaderRequestHandler
						ServiceListRequestHandler
						ServiceQueryRequestHandler
						SubscribeServiceRequestHandler   服务订阅  =  服务获取  + 发布订阅事件
```



获取 全部 Service Instance

```
ServiceInfoHolder  本地service缓存
		FailoverReactor   失败兜底机制

serviceDataIndexes  缓存
singletonRepository 缓存实例工厂
publisherIndexes 实际存放数据
serviceClusterIndex nacos集群数据

获取单个Service Instance

Balancer 负载均衡
```



以下进入源码分析



#### 1. 初始化NamingService

```
private void init(Properties properties) throws NacosException {
        ValidatorUtils.checkInitParam(properties);
        // 命名空间
        this.namespace = InitUtils.initNamespaceForNaming(properties);
        InitUtils.initSerialization();
        InitUtils.initWebRootContext(properties);
        initLogName(properties);
        this.changeNotifier = new InstancesChangeNotifier();
        // 注册服务变更事件（触发器）
        NotifyCenter.registerToPublisher(InstancesChangeEvent.class, 16384);
        // 注册服务变更订阅（监听器）
        NotifyCenter.registerSubscriber(changeNotifier);
        // 本地服务缓存器初始化
        this.serviceInfoHolder = new ServiceInfoHolder(namespace, properties);
        // 和 nacos server发起通信
        this.clientProxy = new NamingClientProxyDelegate(this.namespace, serviceInfoHolder, properties, changeNotifier);
    }
```



#### 2. 服务端 和 客户端 通信

```
NamingClientProxy  实际执行

		NamingHttpClientProxy 长连接(HTTP1.0)
		
				BeatReactor  心跳机制
				
		NamingGrpcClientProxy 短连接(HTTP2.0)
```



#### 3. 服务注册

##### 3.1 客户端

```
@Override
public void registerInstance(String serviceName, String groupName, String ip, int port, String clusterName)
		throws NacosException {
	Instance instance = new Instance();
	instance.setIp(ip);
	instance.setPort(port);
	instance.setWeight(1.0);
	instance.setClusterName(clusterName);
	registerInstance(serviceName, groupName, instance);
}

public void registerService(String serviceName, String groupName, Instance instance) throws NacosException {

	InstanceRequest request = new InstanceRequest(namespaceId, serviceName, groupName,
			NamingRemoteConstants.REGISTER_INSTANCE, instance);
	requestToServer(request, Response.class);
	namingGrpcConnectionEventListener.cacheInstanceForRedo(serviceName, groupName, instance);
}
```



##### 3.2 服务端

InstanceRequestHandler

```
public void registerInstance(Service service, Instance instance, String clientId) {
	Service singleton = ServiceManager.getInstance().getSingleton(service);
	Client client = clientManager.getClient(clientId);
	InstancePublishInfo instanceInfo = getPublishInfo(instance);
	client.addServiceInstance(singleton, instanceInfo);
	client.setLastUpdatedTime();
	// 发布客户端注册事件
	NotifyCenter.publishEvent(new ClientOperationEvent.ClientRegisterServiceEvent(singleton, clientId));
	// 服务服务实例元数据 变更事件
	NotifyCenter.publishEvent(new MetadataEvent.InstanceMetadataEvent(singleton, instanceInfo.getMetadataId(), false));
}
```



##### 3.3 事件发布器

看下**NotifyCenter**的 publishEvent方法

```
private static boolean publishEvent(final Class<? extends Event> eventType, final Event event) {
        if (ClassUtils.isAssignableFrom(SlowEvent.class, eventType)) {
            return INSTANCE.sharePublisher.publish(event);
        }
        
        final String topic = ClassUtils.getCanonicalName(eventType);
        // 根据事件类型获取不同的事件发布器
        EventPublisher publisher = INSTANCE.publisherMap.get(topic);
        if (publisher != null) {
            return publisher.publish(event);
        }
       
        return false;
    }
```



事件发布器有两种

- DefaultSharePublisher 多个事件类型 共享一个事件队列 shareBufferSize = 1024
- DefaultPublisher 每个事件类型对应一个 事件队列 ringBufferSize = 16384

```
void receiveEvent(Event event) {
	final long currentEventSequence = event.sequence();

	if (!hasSubscriber()) {

		return;
	}

// 通知事件的订阅者       
	for (Subscriber subscriber : subscribers) {
		// Whether to ignore expiration events
		if (subscriber.ignoreExpireEvent() && lastEventSequence > currentEventSequence) {

			continue;
		}

		// Because unifying smartSubscriber and subscriber, so here need to think of compatibility.
		// Remove original judge part of codes.
		notifySubscriber(subscriber, event);
	}
}
```



这里事件发布器最终调用的 是 Subscriber 的 onEvent事件，Subscriber 有以下几类

- ClientServiceIndexesManager 客户端事件处理，通常会回调一个 监听客户端同步事件
- DistroClientDataProcessor nacos 集群同步信息
- DistroProtocol 真正执行同步
- NamingMetadataManager
- NamingSubscriberServiceV2Impl 监听的客户端同步信息 处理
- NacosDelayTaskExecuteEngine merge 多个客户端监听事件 并执行
- NacosTaskProcessor process 真正执行同步



这里看下服务注册 的订阅者 也就是 ClientServiceIndexesManager 怎么处理注册节点的

```
 private void addPublisherIndexes(Service service, String clientId) {
        publisherIndexes.computeIfAbsent(service, (key) -> new ConcurrentHashSet<>());
        publisherIndexes.get(service).add(clientId);
        NotifyCenter.publishEvent(new ServiceEvent.ServiceChangedEvent(service, true));
 }
```

可以看到 最终Sercice的信息会存储在publisherIndexes 中，同时发布了一条服务变更的事件 通知到 监听该服务的所有客户端



#### 4. 服务发现

##### 4.1 客户端

```
if (subscribe) {
            serviceInfo = serviceInfoHolder.getServiceInfo(serviceName, groupName, clusterString);
            if (null == serviceInfo) {
                serviceInfo = clientProxy.subscribe(serviceName, groupName, clusterString);
            }
        } else {
            serviceInfo = clientProxy.queryInstancesOfService(serviceName, groupName, clusterString, 0, false);
        }
}  

public ServiceInfo getServiceInfo(final String serviceName, final String groupName, final String clusters) {
       
        String groupedServiceName = NamingUtils.getGroupedName(serviceName, groupName);
        String key = ServiceInfo.getKey(groupedServiceName, clusters);
        if (failoverReactor.isFailoverSwitch()) {
            return failoverReactor.getService(key);
        }
        return serviceInfoMap.get(key);
}    
```



可以看到它先在Client端本地缓存中尝试获取

在查询不到的情况下会向Server端发起请求

```
public ServiceInfo subscribe(String serviceName, String groupName, String clusters) throws NacosException {
        SubscribeServiceRequest request = new SubscribeServiceRequest(namespaceId, groupName, serviceName, clusters,
                true);
        SubscribeServiceResponse response = requestToServer(request, SubscribeServiceResponse.class);
        namingGrpcConnectionEventListener
                .cacheSubscriberForRedo(NamingUtils.getGroupedName(serviceName, groupName), clusters);
        return response.getServiceInfo();
    }
```



这里区分是否需要订阅，需要的话 可以先注册订阅监听事件，然后获取服务实例，这里以需要订阅为例



##### 4.2 服务端

SubscribeServiceRequestHandler

```
   public SubscribeServiceResponse handle(SubscribeServiceRequest request, RequestMeta meta) throws NacosException {
        String namespaceId = request.getNamespace();
        String serviceName = request.getServiceName();
        String groupName = request.getGroupName();
        String app = request.getHeader("app", "unknown");
        String groupedServiceName = NamingUtils.getGroupedName(serviceName, groupName);
        // 请求订阅 / 查询 服务信息
        Service service = Service.newService(namespaceId, groupName, serviceName, true);
        // 订阅者信息
        Subscriber subscriber = new Subscriber(meta.getClientIp(), meta.getClientVersion(), app,
                meta.getClientIp(), namespaceId, groupedServiceName, 0, request.getClusters());
        // 获取服务实例        
        ServiceInfo serviceInfo = handleClusterData(serviceStorage.getData(service),
                metadataManager.getServiceMetadata(service).orElse(null),
                subscriber);
        if (request.isSubscribe()) {
            clientOperationService.subscribeService(service, subscriber, meta.getConnectionId());
        } else {
            clientOperationService.unsubscribeService(service, subscriber, meta.getConnectionId());
        }
        return new SubscribeServiceResponse(ResponseCode.SUCCESS.getCode(), "success", serviceInfo);
    }
```



这里重点看下获取服务实例

```
public ServiceInfo getData(Service service) {
	return serviceDataIndexes.containsKey(service) ? serviceDataIndexes.get(service) : getPushData(service);
}

public ServiceInfo getPushData(Service service) {
	ServiceInfo result = emptyServiceInfo(service);
	if (!ServiceManager.getInstance().containSingleton(service)) {
		return result;
	}
	result.setHosts(getAllInstancesFromIndex(service));
	serviceDataIndexes.put(service, result);
	return result;
}  
```



所以这里可以得出获取全部服务实例的流程就是

1. ServiceInfoHolder 本地service缓存

   FailoverReactor 失败兜底机制

2. serviceDataIndexes 缓存

3. singletonRepository 缓存实例工厂

4. publisherIndexes 实际存放数据

5. serviceClusterIndex nacos集群数据



假如是从全部服务实例中获取其中一个的话，这里有个负载均衡策略，即选择一个最优的

```
String clusterString = StringUtils.join(clusters, ",");
	if (subscribe) {
		ServiceInfo serviceInfo = serviceInfoHolder.getServiceInfo(serviceName, groupName, clusterString);
		if (null == serviceInfo) {
			serviceInfo = clientProxy.subscribe(serviceName, groupName, clusterString);
		}
		return Balancer.RandomByWeight.selectHost(serviceInfo);
	} else {
		ServiceInfo serviceInfo = clientProxy
				.queryInstancesOfService(serviceName, groupName, clusterString, 0, false);
		return Balancer.RandomByWeight.selectHost(serviceInfo);
	}
```



具体负载策略见 Balancer



#### 5. 答疑时间

关于服务注册还有一些问题

**Q1：如何支持高并发注册（异步任务与内存队列设计原理及源码剖析）**

答案：**采用内存队列的方式进行服务注册**

也就是说客户端在把自己的信息注册到Nacos Server的时候，并不是同步把信息写入到注册表中的，而且采取了先写入内存队列中，然后用独立的线程池来消费队列进行注册的。

从源码可看出最终会执行listener.onChange()这个方法，并把Instances传入，然后进行真正的注册逻辑，这里的设计就是为了提高Nacos Server的并发注册量。这里再提一下，在进行队列消费的时候其实最终也是采用的JDK的线程池。



**Q2：注册表如何防止多节点读写并发冲突**

答案：**Copy on write 思想**

updateIps方法中传入了一个List<Instance> ips，然后用ips跟之前注册表中的Instances进行比较，分别得出需要添加、更新、和删除的实例，然后做一些相关的操作，比如Instance的一些属性设置、启动心跳、删除心跳等等，最后把处理后的List<Instance> ips，直接替换内存注册表，这样如果同时有读的请求，其实读取是之前的老注册表的信息，这样就很好的控制了并发读写冲突问题，这个思想就是Copy On Write思想，在JDK源码中并发包里也有一些相关的实现，比如：CopyOnWriteArrayList