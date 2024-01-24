#### 1. 初始化NacosConfigService

```
public NacosConfigService(Properties properties) throws NacosException {
	ValidatorUtils.checkInitParam(properties);

	initNamespace(properties);
	this.configFilterChainManager = new ConfigFilterChainManager(properties);
	ServerListManager serverListManager = new ServerListManager(properties);
	serverListManager.start();

	this.worker = new ClientWorker(this.configFilterChainManager, serverListManager, properties);
	// will be deleted in 2.0 later versions
	agent = new ServerHttpAgent(serverListManager);

}
```



NacosConfigService 包含：

1. IConfigFilterChain --- 拦截链

   IConfigFilter

2. ServerListManager ---获取 和 更新 nacos server 服务集群

3. ClientWorker --- 保持 nacos server的登录有效状态、定期更新本地config 缓存

   securityProxy.login

   cacheMap

   agent.notifyListenConfig();

4. ServerHttpAgent --- 远程ConfigServer调用



#### 2. 获取Config

![img](http://pcc.huitogo.club/c0a9cfa35c8250b0112677bb9cadd0b7)



##### 2.1 客户端

```
private String getConfigInner(String tenant, String dataId, String group, long timeoutMs) throws NacosException {
	group = blank2defaultGroup(group);
	ParamUtils.checkKeyParam(dataId, group);
	ConfigResponse cr = new ConfigResponse();

	cr.setDataId(dataId);
	cr.setTenant(tenant);
	cr.setGroup(group);

	// 优先使用本地配置
	String content = LocalConfigInfoProcessor.getFailover(worker.getAgentName(), dataId, group, tenant);
	if (content != null) {

		cr.setContent(content);
		String encryptedDataKey = LocalEncryptedDataKeyProcessor
				.getEncryptDataKeyFailover(agent.getName(), dataId, group, tenant);
		cr.setEncryptedDataKey(encryptedDataKey);
		configFilterChainManager.doFilter(null, cr);
		content = cr.getContent();
		return content;
	}

	try {
		// 从ServerList中 获取远程nacos server 地址
		ConfigResponse response = worker.getServerConfig(dataId, group, tenant, timeoutMs, false);
		cr.setContent(response.getContent());
		cr.setEncryptedDataKey(response.getEncryptedDataKey());
		configFilterChainManager.doFilter(null, cr);
		content = cr.getContent();

		return content;
	} 
	...
	}
```



优先从本地用户文件夹缓存下获取，比如 C:\Users\huizhang43\nacos\config\fixed-127.0.0.1_8848-dev_nacos\snapshot-tenant\dev\DEFAULT_GROUP



从远程Server端获取

```
ConfigQueryRequest request = ConfigQueryRequest.build(dataId, group, tenant);
            request.putHeader("notify", String.valueOf(notify));
            ConfigQueryResponse response = (ConfigQueryResponse) requestProxy(getOneRunningClient(), request,
                    readTimeouts);

            ConfigResponse configResponse = new ConfigResponse();
```



##### 2.2 服务端

```
        int lockResult = tryConfigReadLock(groupKey);
        
        boolean isBeta = false;
        boolean isSli = false;
        if (lockResult > 0) {
            //FileInputStream fis = null;
            try {
                String md5 = Constants.NULL;
                long lastModified = 0L;
                // 尝试从缓存中获取                
                CacheItem cacheItem = ConfigCacheService.getContentCache(groupKey);
                if (cacheItem != null) {
                    if (cacheItem.isBeta()) {
                        if (cacheItem.getIps4Beta().contains(clientIp)) {
                            isBeta = true;
                        }
                    }
                    String configType = cacheItem.getType();
                    response.setContentType((null != configType) ? configType : "text");
                }
                File file = null;
                ConfigInfoBase configInfoBase = null;
                PrintWriter out = null;
                if (isBeta) {
                    md5 = cacheItem.getMd54Beta();
                    lastModified = cacheItem.getLastModifiedTs4Beta();
                    // 如果nacos server是单机 且 是内嵌的数据库 则直接从数据库中读取
                    if (PropertyUtil.isDirectRead()) {
                        configInfoBase = persistService.findConfigInfo4Beta(dataId, group, tenant);
                    } else {
                    // 否则 从 文件缓存中获取，避免直穿数据库
                        file = DiskUtil.targetBetaFile(dataId, group, tenant);
                    }
                    response.setBeta(true);
               ...     
```



这里对于config文件的存储结构如下：

![img](http://pcc.huitogo.club/7490626b76c4498ecac56e8ac14f2396)