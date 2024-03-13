Dubbo 通过 CacheFilter 过滤器，提供结果缓存的功能，且既可以适用于 Consumer 也可以适用于 Provider 。

通过结果缓存，用于加速热门数据的访问速度，Dubbo 提供声明式缓存，以减少用户加缓存的工作量。



Dubbo 目前提供三种实现：

- `lru` ：基于最近最少使用原则删除多余缓存，保持最热的数据被缓存。
- `threadlocal` ：当前线程缓存，比如一个页面渲染，用到很多 portal，每个 portal 都要去查用户信息，通过线程缓存，可以减少这种多余访问。
- `jcache` ：与 JSR107 集成，可以桥接各种缓存实现。



在源码上主要有CacheFactory、Cache、CacheFilter，其中CacheFactory和CacheFilter是可扩展的

![类图](https://pcc.huitogo.club/z0/20240312734123.png)



#### 1. CacheFilter

`com.alibaba.dubbo.cache.filter.CacheFilter` ，实现 Filter 接口，缓存过滤器实现类。代码如下：

```
 1: @Activate(group = {Constants.CONSUMER, Constants.PROVIDER}, value = Constants.CACHE_KEY)
 2: public class CacheFilter implements Filter {
 3: 
 4:     /**
 5:      * CacheFactory$Adaptive 对象。
 6:      *
 7:      * 通过 Dubbo SPI 机制，调用 {@link #setCacheFactory(CacheFactory)} 方法，进行注入
 8:      */
 9:     private CacheFactory cacheFactory;
10: 
11:     public void setCacheFactory(CacheFactory cacheFactory) {
12:         this.cacheFactory = cacheFactory;
13:     }
14: 
15:     @Override
16:     public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
17:         // 方法开启 Cache 功能
18:         if (cacheFactory != null && ConfigUtils.isNotEmpty(invoker.getUrl().getMethodParameter(invocation.getMethodName(), Constants.CACHE_KEY))) {
19:             // 基于 URL + Method 为维度，获得 Cache 对象。
20:             Cache cache = cacheFactory.getCache(invoker.getUrl().addParameter(Constants.METHOD_KEY, invocation.getMethodName())); // 添加 `method` 属性，是因为 JCache 需要。
21:             if (cache != null) {
22:                 // 获得 Cache Key
23:                 String key = StringUtils.toArgumentString(invocation.getArguments());
24:                 // 从缓存中获得结果。若存在，创建 RpcResult 对象。
25:                 Object value = cache.get(key);
26:                 if (value != null) {
27:                     return new RpcResult(value);
28:                 }
29:                 // 服务调用
30:                 Result result = invoker.invoke(invocation);
31:                 // 若非异常结果，缓存结果
32:                 if (!result.hasException()) {
33:                     cache.put(key, result.getValue());
34:                 }
35:                 return result;
36:             }
37:         }
38:         // 服务调用
39:         return invoker.invoke(invocation);
40:     }
41: 
42: }
```

- 第 18 行：判断方法**开启** Cache 功能。因为，一个服务里，可能只有**部分**方法开启了 Cache 功能。
- 第 20 行：调用 `CacheFactory$Adaptive#getCache(url)` 方法，基于 **URL + Method** 为维度，获得 Cache 对象。
- 第 23 行：调用 `StringUtils#toArgumentString(Object[] args)` 方法，获得 Cache Key 。

- 第 24 至 28 行：调用 `Cache#get(key)` 方法，从缓存中获得结果。若**存在**，创建 RpcResult 对象并返回。
- 第 30 行：调用 `Invoker#invoke(invocation)` 方法，服务调用。
- 第 31 至 34 行：若**非异常**，调用 `Cache#put(key, value)` 方法，缓存**正常的**结果。
- 第 35 行：返回调用结果。
- 第 39 行：若**不使用** Cache 功能，**直接**调用 `Invoker#invoke(invocation)` 方法，服务调用。



#### 2. Cache

`com.alibaba.dubbo.cache.Cache` ，缓存**容器**接口。方法如下：

```
public interface Cache {

    /**
     * 添加键值
     *
     * @param key 键
     * @param value 值
     */
    void put(Object key, Object value);

    /**
     * 获得值
     *
     * @param key 键
     * @return 值
     */
    Object get(Object key);

}
```

- Cache 是个缓存**容器**，内部可以**管理**缓存的键值。



##### 2.1 LRU实现

`com.alibaba.dubbo.cache.support.lru.LruCache` ，实现 Cache 接口，代码如下：

```
public class LruCache implements Cache {

    /**
     * 缓存集合
     */
    private final Map<Object, Object> store;

    public LruCache(URL url) {
        // `"cache.size"` 配置项，设置缓存大小
        final int max = url.getParameter("cache.size", 1000);
        // 创建 LRUCache 对象
        this.store = new LRUCache<Object, Object>(max);
    }

    @Override
    public void put(Object key, Object value) {
        store.put(key, value);
    }

    @Override
    public Object get(Object key) {
        return store.get(key);
    }

}
```

- `"cache.size"` 配置项，设置缓存**大小**。
- 基于 `com.alibaba.dubbo.common.utils.LRUCache` 实现。



##### 2.2 ThreadLocal 实现

`com.alibaba.dubbo.cache.support.threadlocal.ThreadLocalCache` ，实现 Cache 接口，代码如下：

```
public class ThreadLocalCache implements Cache {

    private final ThreadLocal<Map<Object, Object>> store; // 线程变量

    public ThreadLocalCache(URL url) {
        this.store = new ThreadLocal<Map<Object, Object>>() {

            @Override
            protected Map<Object, Object> initialValue() {
                return new HashMap<Object, Object>();
            }

        };
    }

    @Override
    public void put(Object key, Object value) {
        store.get().put(key, value);
    }

    @Override
    public Object get(Object key) {
        return store.get().get(key);
    }

}
```

- 基于 ThreadLocal 实现，相当于一个线程，一个 ThreadLocalCache 对象。
- ThreadLocalCache 目前没有过期或清理机制，所以**需要注意**。



##### 2.3 JCache 实现

`com.alibaba.dubbo.cache.support.jcache.JCache` ，实现 Cache 接口，代码如下：

```
public class JCache implements com.alibaba.dubbo.cache.Cache {

    private final Cache<Object, Object> store;

    public JCache(URL url) {
        // 获得 Cache Key
        String method = url.getParameter(Constants.METHOD_KEY, "");
        String key = url.getAddress() + "." + url.getServiceKey() + "." + method;
        // `"jcache"` 配置项，为 Java SPI 实现的全限定类名
        // jcache parameter is the full-qualified class name of SPI implementation
        String type = url.getParameter("jcache");

        // 基于类型，获得 javax.cache.CachingProvider 对象，
        CachingProvider provider = type == null || type.length() == 0 ? Caching.getCachingProvider() : Caching.getCachingProvider(type);
        // 获得 javax.cache.CacheManager 对象
        CacheManager cacheManager = provider.getCacheManager();
        // 获得 javax.cache.Cache 对象
        Cache<Object, Object> cache = cacheManager.getCache(key);
        // 不存在，则进行创建
        if (cache == null) {
            try {
                // 设置 Cache 配置项
                // configure the cache
                MutableConfiguration config =
                        new MutableConfiguration<Object, Object>()
                                // 类型
                                .setTypes(Object.class, Object.class)
                                // 过期策略，按照写入时间过期。通过 `"cache.write.expire"` 配置项设置过期时间，默认为 1 分钟。
                                .setExpiryPolicyFactory(CreatedExpiryPolicy.factoryOf(new Duration(TimeUnit.MILLISECONDS, url.getMethodParameter(method, "cache.write.expire", 60 * 1000))))
                                .setStoreByValue(false)
                                // 设置 MBean
                                .setManagementEnabled(true)
                                .setStatisticsEnabled(true);
                // 创建 javax.cache.Cache 对象
                cache = cacheManager.createCache(key, config);
            } catch (CacheException e) {
                // 初始化 cache 的并发情况
                // concurrent cache initialization
                cache = cacheManager.getCache(key);
            }
        }

        this.store = cache;
    }

    @Override
    public void put(Object key, Object value) {
        store.put(key, value);
    }

    @Override
    public Object get(Object key) {
        return store.get(key);
    }

}
```



#### 3. CacheFactory

`com.alibaba.dubbo.cache.CacheFactory` ，Cache 工厂**接口**。方法如下：

```
@SPI("lru")
public interface CacheFactory {

    /**
     * 获得缓存对象
     *
     * @param url URL 对象
     * @return 缓存对象
     */
    @Adaptive("cache")
    Cache getCache(URL url);

}
```

- `@SPI("lru")` 注解，Dubbo SPI **拓展点**，默认为 `"lru"` 。
- `@Adaptive("cache")` 注解，基于 Dubbo SPI Adaptive 机制，加载对应的 Cache 实现，使用 `URL.cache` 属性。



`com.alibaba.dubbo.cache.support.AbstractCacheFactory` ，Cache 工厂**抽象类**。代码如下：

```
public abstract class AbstractCacheFactory implements CacheFactory {

    /**
     * Cache 集合
     *
     * key：URL
     */
    private final ConcurrentMap<String, Cache> caches = new ConcurrentHashMap<String, Cache>();

    @Override
    public Cache getCache(URL url) {
        // 获得 Cache 对象
        String key = url.toFullString();
        Cache cache = caches.get(key);
        // 不存在，创建 Cache 对象，并缓存
        if (cache == null) {
            caches.put(key, createCache(url));
            cache = caches.get(key);
        }
        return cache;
    }

    /**
     * 创建 Cache 对象
     *
     * @param url URL
     * @return Cache 对象
     */
    protected abstract Cache createCache(URL url);

}
```



