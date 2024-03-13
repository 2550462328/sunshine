#### 1. Dubbo官方推荐的异常处理方式是什么？

在dubbo官方文档的服务化最佳实践中,推荐的处理方式如下:

![img](https://pcc.huitogo.club/z0/20161228230546795)



#### 2. Dubbo处理异常的逻辑是什么样的？为什么要这样处理？

dubbo的异常处理类是com.alibaba.dubbo.rpc.filter.ExceptionFilter 类，大致源码如下：

```
 1: @Activate(group = Constants.PROVIDER)
 2: public class ExceptionFilter implements Filter {
 3: 
 4:     private final Logger logger;
 5: 
 6:     public ExceptionFilter() {
 7:         this(LoggerFactory.getLogger(ExceptionFilter.class));
 8:     }
 9: 
10:     public ExceptionFilter(Logger logger) {
11:         this.logger = logger;
12:     }
13: 
14:     @Override
15:     public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
16:         try {
17:             // 服务调用
18:             Result result = invoker.invoke(invocation);
19:             // 有异常，并且非泛化调用
20:             if (result.hasException() && GenericService.class != invoker.getInterface()) {
21:                 try {
22:                     Throwable exception = result.getException();
23: 
24:                     // directly throw if it's checked exception
25:                     // 如果是checked异常，直接抛出
26:                     if (!(exception instanceof RuntimeException) && (exception instanceof Exception)) {
27:                         return result;
28:                     }
29:                     // directly throw if the exception appears in the signature
30:                     // 在方法签名上有声明，直接抛出
31:                     try {
32:                         Method method = invoker.getInterface().getMethod(invocation.getMethodName(), invocation.getParameterTypes());
33:                         Class<?>[] exceptionClassses = method.getExceptionTypes();
34:                         for (Class<?> exceptionClass : exceptionClassses) {
35:                             if (exception.getClass().equals(exceptionClass)) {
36:                                 return result;
37:                             }
38:                         }
39:                     } catch (NoSuchMethodException e) {
40:                         return result;
41:                     }
42: 
43:                     // 未在方法签名上定义的异常，在服务器端打印 ERROR 日志
44:                     // for the exception not found in method's signature, print ERROR message in server's log.
45:                     logger.error("Got unchecked and undeclared exception which called by " + RpcContext.getContext().getRemoteHost()
46:                             + ". service: " + invoker.getInterface().getName() + ", method: " + invocation.getMethodName()
47:                             + ", exception: " + exception.getClass().getName() + ": " + exception.getMessage(), exception);
48: 
49:                     // 异常类和接口类在同一 jar 包里，直接抛出
50:                     // directly throw if exception class and interface class are in the same jar file.
51:                     String serviceFile = ReflectUtils.getCodeBase(invoker.getInterface());
52:                     String exceptionFile = ReflectUtils.getCodeBase(exception.getClass());
53:                     if (serviceFile == null || exceptionFile == null || serviceFile.equals(exceptionFile)) {
54:                         return result;
55:                     }
56:                     // 是JDK自带的异常，直接抛出
57:                     // directly throw if it's JDK exception
58:                     String className = exception.getClass().getName();
59:                     if (className.startsWith("java.") || className.startsWith("javax.")) {
60:                         return result;
61:                     }
62:                     // 是Dubbo本身的异常，直接抛出
63:                     // directly throw if it's dubbo exception
64:                     if (exception instanceof RpcException) {
65:                         return result;
66:                     }
67: 
68:                     // 否则，包装成RuntimeException抛给客户端
69:                     // otherwise, wrap with RuntimeException and throw back to the client
70:                     return new RpcResult(new RuntimeException(StringUtils.toString(exception)));
71:                 } catch (Throwable e) {
72:                     logger.warn("Fail to ExceptionFilter when called by " + RpcContext.getContext().getRemoteHost()
73:                             + ". service: " + invoker.getInterface().getName() + ", method: " + invocation.getMethodName()
74:                             + ", exception: " + e.getClass().getName() + ": " + e.getMessage(), e);
75:                     return result;
76:                 }
77:             }
78:             // 返回
79:             return result;
80:         } catch (RuntimeException e) {
81:             logger.error("Got unchecked and undeclared exception which called by " + RpcContext.getContext().getRemoteHost()
82:                     + ". service: " + invoker.getInterface().getName() + ", method: " + invocation.getMethodName()
83:                     + ", exception: " + e.getClass().getName() + ": " + e.getMessage(), e);
84:             throw e;
85:         }
86:     }
87: 
88: }
```



总结如下：

1. 如果provider实现了GenericService接口,直接抛出

2. 如果是checked异常，直接抛出

3. 在方法签名上有声明，直接抛出

4. 异常类和接口类在同一jar包里，直接抛出

5. 是JDK自带的异常，直接抛出

6. 是Dubbo本身的异常，直接抛出

7. 否则，包装成RuntimeException抛给客户端


网上有些文章对7)的处理有疑问,不理解原因,其实就是为了防止客户端反序列化失败.前面几种情况都能保证反序列化正常。



#### 3. 抛出自定义异常有哪些方式？

抛出自定义异常其实就是使用上面2中的逻辑.所以相对应的有以下几种方式:

1. provider实现GenericService接口.(我没试过这种方式,应该是要自己实现invoke( ) 方法 , 网上说直接把invoke()方法废弃掉不知道是怎么处理的,直接返回null肯定是不行的)

2. 自定义异常声明为checked异常(这没啥说了,不过一般自定义异常都是unchecked)

3. 在方法签名上声明抛出异常(这种基本上所有接口都要写,麻烦)

4. 异常类和接口类在同一jar包里(存在链式调用时,这种可能不适用)

5. 自定义异常的包名以java.或javax.开头(dubbo判断jdk自带异常的条件,一般项目都有自己的命名规范,这样干的估计很少)




除了上面对应的,还可以用一种奇葩的方式,直接去掉异常的filter,如下:

```
<dubbo:provider filter=“-exception” />
```



#### 4. 在dubbo:provider中设置filter=“-exception”, 去掉异常的filter会怎么样?

我在处理自定义异常的时候,觉得3中提到前5种方式都不太适合,所以使用了这种方式.

这种方式对provider中抛出的异常不做任何处理,直接返回给consumer,会遇到一些奇怪的问题;



例如,在provider中有以下代码:

```
		String respone = "avs";
		JSONObject resultjson = JSONObject.fromObject(respone);
```



这段代码会抛出 net.sf.json.JSONException ,但是在provider中不会打印任何异常信息.

而在consumer中,会报出下面的错误:

```
com.alibaba.com.caucho.hessian.io.HessianFieldException: org.apache.commons.lang.exception.NestableRuntimeException.delegate: 'org.apache.commons.lang.exception.NestableDelegate' could not be instantiated
	at com.alibaba.com.caucho.hessian.io.JavaDeserializer.logDeserializeError(JavaDeserializer.java:671)
	at com.alibaba.com.caucho.hessian.io.JavaDeserializer$ObjectFieldDeserializer.deserialize(JavaDeserializer.java:400)
	at com.alibaba.com.caucho.hessian.io.JavaDeserializer.readObject(JavaDeserializer.java:233)
	at com.alibaba.com.caucho.hessian.io.JavaDeserializer.readObject(JavaDeserializer.java:157)
	at com.alibaba.com.caucho.hessian.io.SerializerFactory.readObject(SerializerFactory.java:397)
	at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObjectInstance(Hessian2Input.java:2070)
	at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:2005)
	at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:1990)
```

这是个因为异常传递引起的错误,原异常信息也丢失了,无法定位问题.

为了解决这个问题,想到的方案是在provider中,把所有入口方法都进行catch处理,**转换为自定义异常**。



#### 5. 最终采用的异常处理方案

抛出一个自定义异常有这么麻烦吗 主要原因是dubbo没有支持

既然这样,我们把dubbo变的支持不就可以了

是的.把源码改一下就OK了.如下图:

![img](https://pcc.huitogo.club/z0/20161229210802425)

在异常处理这里,加上自定义异常处理的代码.

或者直接将112行的RuntimeException替换成自己的自定义异常!

修改源码后,可以替换maven仓库的jar,或者是在自己项目里面建一个ExceptionFilter,包名和dubbo的相同,用来覆盖掉dubbo的

(如果替换112行为自定义异常,则要引入自定义的包等).

这样就从根本上解决了异常处理的问题.后续有其他问题,也可以直接修改.
