看一下Gateway的流程图

![img](http://pcc.huitogo.club/176917e1932cac56d350b06078ca8b0f)



在上述过程中，客户端向Spring Cloud Gateway发出请求。 如果Gateway Handler Mapping**确定请求与路由匹配**（这个时候就用到predicate），则将其发送到Gateway web handler处理。 Gateway web handler处理请求时会**经过一系列的过滤器链**。过滤器链被虚线划分的原因是过滤器链可以在发送代理请求之前或之后执行过滤逻辑。先执行所有“pre”过滤器逻辑，然后进行代理请求。在发出代理请求之后，收到代理服务的响应之后执行“post”过滤器逻辑。这跟zuul的处理过程很类似。

**在执行所有“pre”过滤器逻辑时，往往进行了鉴权、限流、日志输出等功能，以及请求头的更改、协议的转换**；**转发之后收到响应之后，会执行所有“post”过滤器的逻辑，在这里可以响应数据进行了修改，比如响应头、协议的转换等。**



#### 1. Predict（路由转发）

在上面的处理过程中，有一个重要的点就是讲请求和路由进行匹配，这时候就需要用到predicate，它是决定了一个请求走哪一个路由。

Spring Cloud Gateway内置了许多Predict,这些Predict的源码在org.springframework.cloud.gateway.handler.predicate包中，predicate包中的子类图如下：

![img](http://pcc.huitogo.club/bd40a0c2de0e6fdc1b3fafebb20b122a)



对应的构建Routeloader时：

```
1.  @Bean  

2.  public RouteLocator myRoutes(RouteLocatorBuilder builder) {  

3.      String httpUri = "http://httpbin.org:80";  

4.      return builder  

5.          .routes()  

6.          .route(  

7.              "route1",  

8.              p -> p.after(ZonedDateTime.now()).negate().or().header("hui", "hui").or().cookie("hui", "hui").and().path("/get").method(Method.GET).uri(httpUri))  

9.  .build();

10. }  
```



上述中.after指时间在当前时间之后，对应的还有.before和.between

- .negate()代表否的意思，这里指不在当前时间之后，也就是当前时间之前
- .or代表或的意思，用来连接两个条件语句，对应的还有.and
- .header判断头部信息
- .cookie判断cookie信息
- .path判断请求路径
- .method判断请求方式
- .query 判断请求参数

满足以上条件后在将请求转发到http://httpbin.org:80



#### 2. Filter（权限校验）

Predict决定了请求由哪一个路由处理，在路由处理之前，需要经过“pre”类型的过滤器处理，处理返回响应之后，可以由“post”类型的过滤器处理。

- “pre”类型的过滤器可以做**参数校验**、**权限校验**、**流量监控**、**日志输出**、**协议转换**等
- “post”类型的过滤器中可以做**响应内容**、**响应头的修改**，**日志的输出**，**流量监控**等

与zuul不同的是，filter除了分为“pre”和“post”两种方式的filter外，**在Spring Cloud Gateway中，filter从作用范围可分为另外两种**，一种是针对于**单个路由**的gateway filter，它在配置文件中的写法同predict类似；另外一种是针对于**所有路由**的global gateway filer



##### 2.1 单个路由的gateway filter

单个路由的gateway filter 需要通过spring.cloud.routes.filters 配置在具体路由下，只作用在当前路由上或通过spring.cloud.default-filters配置在全局，作用在所有路由上。



**1）过滤器工厂和过滤器的关系？**

网关过滤器工厂接口有多个实现类，**在每个 GatewayFilterFactory实现类（过滤器工厂）的 apply（ T config） 方法里，都声明了一个实现 GatewayFilter的内部类（过滤器）**。



**2）过滤器和过滤器工厂的种类？**

过滤器 有 20 多个 实现 类， 包括 头部 过滤器、 路径 类 过滤器、 Hystrix 过滤器和 变更 请求 URL 的 过滤器， 还有 参数 和 状态 码 等 其他 类型 的 过滤器。

内置的过滤器工厂有22个实现类，包括 头部过滤器、路径过滤器、Hystrix 过滤器、请求URL变更过滤器，还有参数和状态码等其他类型的过滤器。根据过滤器工厂的用途来划分，可以分为以下几种：Header、Parameter、Path、Body、Status、Session、Redirect、Retry、RateLimiter和Hystrix。



下图是Gateway内置的过滤器工厂分类

![img](http://pcc.huitogo.club/f714d49049e3979e9e4eeb4556556791)



**3）如何使用过滤器工厂？**

常用的过滤器如AddRequestHeader GatewayFilter Factory

```
1.  @Bean  

2.  public RouteLocator myRoutes(RouteLocatorBuilder builder) {  

3.     return builder.routes()  

4.      .route(p -> p  

5.          .path("/get")  

6.          .filters(f -> f.addRequestHeader("Hello", "World"))  

7.          .uri("http://httpbin.org:80"))  

8.      .build();  

9.  }  
```



上述实现的功能就是对请求路径为/get的请求的header中添加hello:world键值对，然后将请求转发到http://httpbin.org:80



还有一种RewritePath GatewayFilter Factory，这是zuul没有的功能，对路径进行重写。



示例代码如下：

```
1.  @Bean  

2.  public RouteLocator myRoutes(RouteLocatorBuilder builder) {  

3.      String httpUri = "http://httpbin.org:80";  

4.      return builder  

5.          .routes()  

6.          // 对localhost:8777/rewrite/index的请求会被转发到https://blog.csdn.net/index  

7.          .route(  

8.              "route2",  

9.              p -> p.path("/rewrite/**").filters(f -> f.rewritePath("/rewrite/(?<segment>.*)", "/${segment}"))  

10.                 .uri("https://blog.csdn.net"))  

11.         .build();  

12. } 
```



上述实现的功能就是对localhost:8777/rewrite/index的请求会被转发到https://blog.csdn.net/index。


**4）自定义单个过滤器**

除了Spring Cloud Gateway内置的过滤器外，还可以**自定义过滤器**

过滤器需要实现GatewayFilter和Ordered2个接口，示例代码如下：

```
1.  public class GateWayFilter implements GatewayFilter, Ordered {  

2.      private static final String REQUEST_BEGIN_TIME = "requestTimeBegin";  

3.      @Override  

4.      public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {  

5.          //此处相当于zuul中的"pre"阶段  

6.          exchange.getAttributes().put(REQUEST_BEGIN_TIME, System.currentTimeMillis());  

7.          return chain.filter(exchange).then(  

8.              Mono.fromRunnable(() -> {  

9.                  //此处相当于zuul中的"post"阶段  

10.                 Long startTime = exchange.getAttribute(REQUEST_BEGIN_TIME);  

11.                 if (startTime != null) {  

12.                     System.out.println("执行" + exchange.getRequest().getURI().getRawPath() + "耗时："  + (System.currentTimeMillis() - startTime) + "ms");  

14.                 }  

15.             }));  

16.     }  


17.     // 值越大则优先级越低  

18.     @Override  

19.     public int getOrder() {  

20.         return 0;  

21.     }  

22. }  
```



上述自定义过滤器实现了记录请求处理的时间并输出。

构建RouteLocator，加入自定义的filter

```
1.  @Bean  

2.  public RouteLocator myRoutes(RouteLocatorBuilder builder) {  

3.      String httpUri = "http://httpbin.org:80";  

4.      return builder  

5.          .routes()  

6.          // 让请求“/get”请求都转发到“http://httpbin.org/get”  

7.          .route(  

8.              "route1",  

9.              p -> p.path("/get").filters(f -> f.filter(new GateWayFilter()))  

10.                 .uri(httpUri))  

11.         .build();  

12. }  
```



##### 2.2 全局filter

全局过滤器，不需要在配置文件中配置，作用在所有的路由上，最终通过GatewayFilterAdapter包装成GatewayFilterChain可识别的过滤器，它为请求业务以及路由的URI转换为真实业务服务的请求地址的核心过滤器，不需要配置，系统初始化时加载，并作用在每个路由上。



Spring Cloud Gateway框架内置的GlobalFilter如下：

![img](http://pcc.huitogo.club/a51a7c05f1d481a08df800f32cd90ee6)



Gateway内置的全局过滤器默认都是生效的，可以满足大部分需求，下面用自定义的全局过滤器来进行权限验证，自定义全局过滤器需要实现GlobalFilter接口。



示例代码如下：

```
1.  public class GlobalFilter implements org.springframework.cloud.gateway.filter.GlobalFilter, Ordered {  

2.      Logger logger = LoggerFactory.getLogger(TokenFilter.class);  

3.      @Override  

4.      public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {  

5.          String token = exchange.getRequest().getQueryParams().getFirst("token");  

6.           可以使用token做权限验证  

7.          if (token == null || token.isEmpty()) {  

8.              logger.info("token is empty...");  

9.              exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);  

10.             return exchange.getResponse().setComplete();  

11.         }  

12.         return chain.filter(exchange);  

13.     }  

14.     @Override  

15.     public int getOrder() {  

16.         return -100;  

17.     }  

18. }  
```



上述功能实现了对项目中的所有请求路径都进行权限验证，判断请求参数中的token值。

然后需要将TokenFilter在工程的启动类中注入到Spring Ioc容器中。



##### 2.3 自定义过滤器工厂

过滤器都是由过滤器工厂处理的，我们可以自定义过滤器工厂来实现两个或以上的过滤器功能。



GatewayFilterFactory的继承关系如下：

![img](http://pcc.huitogo.club/7671a0071d25c0ee538644262357b601)



由 GatewayFilterFactory 类图可知，GatewayFilterFactory主要有三个抽象类

- **AbstractGatewayFilterFactory**

- **AbstractNameValueGatewayFilterFactory**(继承自AbstractGatewayFilterFactory)

  该抽象类用于提供通用的方法给键值对参数类型的网关过滤器，接收两个参数，如 AddRequestHeader 这种类型的过滤器

- **AbstractChangeRequestUriGatewayFilterFactory**(继承自AbstractGatewayFilterFactory)

  改变请求的 URI 过滤器，该过滤器通过方法 determineRequestUri（ServerWebExchange，T）实现改变请求URI的逻辑



下面我们自定义过滤器工厂，将请求的日志打印出来，需要使用一个参数，这时可以参照RedirectToGatewayFilterFactory的写法



示例代码如下：

```
1.  public class RequestTimeGatewayFilterFactory extends AbstractGatewayFilterFactory<RequestTimeGatewayFilterFactory.Config> {  

2.      private static final Log log = LogFactory.getLog(GatewayFilter.class);  

3.      private static final String REQUEST_TIME_BEGIN = "requestTimeBegin";  

4.      private static final String KEY = "withParams";  

5.      //一定要调用下父类的构造器把Config类型传过去  

6.      public RequestTimeGatewayFilterFactory() {  

7.          super(Config.class);  

8.      }  

9.      @Override  

10.     public GatewayFilter apply(Config config) {  

11.         return (exchange, chain) -> {  

12.             exchange.getAttributes().put(REQUEST_TIME_BEGIN, System.currentTimeMillis());  

13.             return chain.filter(exchange).then(  

14.                     Mono.fromRunnable(() -> {  

15.                         Long startTime = exchange.getAttribute(REQUEST_TIME_BEGIN);  

16.                         if (startTime != null) {  

17.                             StringBuilder sb = new StringBuilder(exchange.getRequest().getURI().getRawPath())  

18.                                     .append(": ")  

19.                                     .append(System.currentTimeMillis() - startTime)  

20.                                     .append("ms");  

21.                             if (config.isWithParams()) {  

22.                                 sb.append(" params:").append(exchange.getRequest().getQueryParams());  

23.                             }  

24.                             log.info(sb.toString());  

25.                         }  

26.                     })  

27.             );  

28.         };  

29.     }  

30.     @Override  

31.     public List<String> shortcutFieldOrder() {  

32.         return Arrays.asList(KEY);  

33.     }  

34.     public static class Config {  

35.         private boolean withParams;  

36.         public boolean isWithParams() {  

37.             return withParams;  

38.         }  

39.         public void setWithParams(boolean withParams) {  

40.             this.withParams = withParams;  

41.         }  

42.     }  

43. } 
```



在上面的代码中 apply(Configconfig)方法内创建了一个GatewayFilter的匿名类，具体的实现逻辑跟之前一样，只不过加了是否打印请求参数的逻辑，而这个逻辑的开关是config.isWithParams()。静态内部类类Config就是为了接收那个boolean类型的参数服务的，里边的变量名可以随意写，但是要重写List shortcutFieldOrder()这个方法。

需要注意的是，在类的构造器中一定要调用下父类的构造器把Config类型传过去，否则会报ClassCastException



再注入到Spring中去

```
1.  @Bean  

2.  public RequestTimeGatewayFilterFactory elapsedGatewayFilterFactory() {  

3.      return new RequestTimeGatewayFilterFactory();  

4.  }  
```



在application.yml中配置过滤器属性

```
1.   #自定义GateWayFactory  

2.   #RequestTime为自定义GateWayFilterFactory的前缀 true/false 设置 withParams  

3.     - id: elapse_route  

4.            uri: http://httpbin.org:80/get  

5.   - RequestTime=true  

6.     predicates:  

7.  - Path=/requestTime/{segment} 
```



#### 3. 限流控制

在高并发的系统中，往往需要在系统中做限流，一方面是为了防止大量的请求使服务器过载，导致服务不可用，另一方面是为了防止网络攻击。

常见的限流方式，比如Hystrix适用线程池隔离，超过线程池的负载，走熔断的逻辑。在一般应用服务器中，比如tomcat容器也是通过限制它的线程数来控制并发的；也有通过时间窗口的平均速度来控制流量。常见的限流纬度有比如通过Ip来限流、通过uri来限流、通过用户访问频次来限流。

一般限流都是在网关这一层做，比如Nginx、Openresty、kong、zuul、Spring Cloud Gateway等；也可以在应用层通过Aop这种方式去做限流。

限流算法详细参考另一篇，这里讨论的是用Spring Cloud Gateway实现限流

Spring Cloud Gateway官方提供了**RequestRateLimiterGatewayFilterFactory**这个类，适用**Redis和lua脚本实现了令牌桶的方式**。具体实现逻辑在RequestRateLimiterGatewayFilterFactory类中。



实现步骤如下：

1. 添加spring-cloud-starter-gateway、spring-boot-starter-data-redis-reactive依赖。
2. 配置application.yml中redis和配置了RequestRateLimiter的限流过滤器

```
1.  spring:  

2.    application.name: eureka-gatewayroutes  

3.    # redis 库  

4.    redis:  

5.      host: localhost  

6.      port: 6379  

7.      database: 0  

8.    # spring cloud  

9.    cloud:  

10.     gateway:   

11.       routes:  

12.         filters:  

13.         -  name: RequestRateLimiter  

14.            args:  

15.              # 用于限流的键的解析器的Bean对象的名字  

16.              key-resolver: '#{@hostAddrKeyResolver}'  

17.              # 每秒填充的平均速率  

18.              redis-state-limiter.replenishRate: 1  

19.              #令牌桶总容量  

20.              redis-state-limiter.busrstCapcaty: 3  
```



3. 从Host角度进行限流，需要实现KeyResolver接口

```
1.  public class HostAddrKeyResolver implements KeyResolver {  

2.      @Override  

3.      public Mono<String> resolve(ServerWebExchange exchange) {  

4.          // 根据ip限流  

5.          return Mono.just(exchange.getRequest().getRemoteAddress().getAddress().getHostAddress());  

6.          // 根据url限流  

7.          // return Mono.just(exchange.getRequest().getURI().getPath());  

8.          // 根据传入的user参数限流  

9.          // return exchange -> Mono.just(exchange.getRequest().getQueryParams().getFirst("user"));  

10.     }  

11. }  
```



4. 将自定义的HostAddrKeyResolver注入到Spring中

```
1.  public HostAddrKeyResolver hostAddrKeyResolver() {  

2.      return new HostAddrKeyResolver();  

3.  } 
```