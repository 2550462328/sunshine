
![img](http://pcc.huitogo.club/9472548801489f5357f172fc5453fd2a)



#### 1. 入口

```
AbstractAutoProxyCreator

                        postProcessAfterInitialization

                        postProcessBeforeInstantiation
```

![img](http://pcc.huitogo.club/86b2c42fed20357de0ff9d5132ec60b7)



#### 2. 获取Adivsor

```
Advisor ReflectiveAspectJAdvisorFactory.getAdvisors()

  ReflectiveAspectJAdvisorFactory.getAdvice()
```

Spring中有五种增强：BeforeAdvide（前置增强）、AfterAdvice（后置增强）、ThrowsAdvice（异常增强）、RoundAdvice（环绕增强 = 前置增强 + 后置增强）、IntroductionAdvice（引入增强）

PointcutAdvisor：对targetObject 的 method 使用 pointexpression 判断,，拦截成功，加入MethodIntercptor调用链 - **对一个类已知的方法进行增强** 

IntroductionAdvisor：针对 targetObject 的 class， 增加（不是实现）一个或多个接口，在代理类层面赋予targetObject新的接口能力 在代理类调用方法时会对method进行判断 如果匹配增强接口 则执行MethodInterceprot – **对一个类添加新的接口和方法进行增强**

```
1.    class MyIntroductionInterceptor implements IntroductionInterceptor, AddtionalInterface, CallbackInterface { 

2.      @Override 

3.      public void enhance() { 

4.        System.out.println("i am a enhance proxy method"); 

5.      } 

6.     

7.      @Override 

8.      public void callback() { 

9.        System.out.println("i am just a little callback function"); 

10.     } 

11.    

12.     @Override 

13.     public Object invoke(MethodInvocation invocation) throws Throwable { 

14.   //    必须要匹配指定接口的方法才能增强 

15.       if (implementsInterface(invocation.getMethod().getDeclaringClass())) { 

16.         System.out.println("surprise, i have through the filter and now begin run"); 

17.         return invocation.getMethod().invoke(this, invocation.getArguments()); 

18.       } 

19.       return invocation.proceed(); 

20.     } 

21.    

22.     @Override 

23.     public boolean implementsInterface(Class<?> intf) { 

24.       return intf.isAssignableFrom(AddtionalInterface.class) || intf.isAssignableFrom(CallbackInterface.class); 

25.     } 

26.   } 

27.    

28.     public static void main(String[] args) { 

29.       ProxyFactory proxyFactory = new ProxyFactory(new People()); 

30.       // CGLIB 代理 因为是基于类增强 （不一定有实现的接口） 

31.       proxyFactory.setProxyTargetClass(true); 

32.    

33.       Advice advice = new MyIntroductionInterceptor(); 

34.       DefaultIntroductionAdvisor advisor = new DefaultIntroductionAdvisor((DynamicIntroductionAdvice)advice, AddtionalInterface.class); 

35.   //    可以声明多个接口 只要IntroductionInterceptor实现和 匹配该接口 

36.       advisor.addInterface(CallbackInterface.class); 

37.    

38.       proxyFactory.addAdvisor(advisor); 

39.       // 在不影响People 的情况下，给它的代理类新增了 接口 功能 

40.       AddtionalInterface proxy = (AddtionalInterface) proxyFactory.getProxy(); 

41.       proxy.enhance(); 

42.    

43.       CallbackInterface proxy2 = (CallbackInterface) proxyFactory.getProxy(); 

44.       proxy2.callback(); 

45.        

46.   //    不影响targerObject 原有方法的逻辑 

47.       People people = (People) proxyFactory.getProxy(); 

48.       people.say(); 

49.     } 
```



#### 3. 过滤Advisor

AopUtil.canApply()



#### 4. Advisor排序

AnnotationAwareOrderComparator



**Q1：为什么ExposeInvocationInterceptor放在所有MethodInterceptor的前面？**

ExposeInvocationInterceptor 会将当前代理类的 MethodInvocation 放在 ThreadLocal中 以备使用，为什么放在第一位就是 希望后面所有的MethodInterceptor都能访问到 这个全局的MethodInvocation

```
1.    ExposeInvocationInterceptor.invoke()

2.     

3.    public Object invoke(MethodInvocation mi) throws Throwable { 

4.      MethodInvocation oldInvocation = invocation.get(); 

5.      invocation.set(mi); 

6.      try { 

7.       // 在代理类的 advice chain 执行期间 都能用ExposeInvocationInterceptor.currentInvocation() 拿到 mi 的信息 

8.        return mi.proceed(); 

9.      } 

10.     finally { 

11.       invocation.set(oldInvocation); 

12.     } 

13.   } 

14.    

15.   使用案例

16.    @Around("printSysLog()") 

17.    public Object saveSysLog(ProceedingJoinPoint proceedingJoinPoint) throws Throwable { 

18.      MethodInvocation invocation = ExposeInvocationInterceptor.currentInvocation(); 

19.      ...... 

20.   } 
```

![img](http://pcc.huitogo.club/1fae6cfdf98dd6eafa8e1d6a82c5c6ac)



#### 5. 创建代理

所有代理工厂类 继承自 ProxyConfig



proxyTargetClass：是否要直接通过类进行代理（无论接口 还是 类） optimize：是否对代理类进行优化，比如不允许代理类创建之后，在Advice发生改变后跟着改变 opaque：是否阻止将代理类变成Advised，变成Advised可以人工干预（可以查询代理 类的属性） exposeProxy：是否将代理类自身注入自身的ThreadLocak（解决代理类调用自身代理方 法的问题），为true时 opaque 不生效 frozen： 是否冻结代理类对象，即代理类生成后，任何变更的Advice都对它无效



AopProxyFactory.createAopProxy

ProxyProcessorSupport. evaluateProxyInterfaces 确定 有无需要代理的接口（jdk 还是 cglib）如果是jdk 则 装载接口信息到ProxyFactory



#### 6. 链式执行

Advice -> 对应 MethodInterceptor 怎么执行 Advisor -> Advice 的适配器 什么时候执行（切入点） + 怎么执行 Advised -> 代理类容器 = Interceptors + other advice + Advisors + the proxied interfaces

AdvisedSupport.getInterceptorsAndDynamicInterceptionAdvice 获取代理方法上 真正执行的 拦截器链 （一堆Advisor）

AdvisorAdapter. getInterceptor(Advisor advisor) 转换Advisor 到对应的 MethodInteceptor AdvisorAdapterRegistry. registerAdvisorAdapter 注册AdvisorAdapter （so 可以自定义）

![img](http://pcc.huitogo.club/015795c232e14e34a0b92805cc9197cf)



最后执行的时候是执行 Advisor 所对应的 MethodInteceptor

BeforeAdvice、AfterAdvice、AbstractAspectJAdvice

![img](http://pcc.huitogo.club/843e5a56e8a4bf51cbb54107aa926c9c)



#### 7. 扩展

##### 7.1 DynamicDataSourceAnnotationAdvisor 动态数据源 

![img](http://pcc.huitogo.club/ea8f4fbd1c883670e77b451abfbd088a)



1）定义切面（拦截哪些类 或 方法）

也就是带DS 注解的

```
DynamicDataSourceAnnotationAdvisor. buildPointcut()
          AnnotatedElementUtils.hasAnnotation(clazz,DS.class)
          AnnotatedElementUtils.hasAnnotation(method,DS.class)
```



2）定义通知（拦截之后做什么）

```
DynamicDataSourceAnnotationInterceptor.invoke()
     DynamicDataSourceContextHolder. LOOKUP_KEY_HOLDER  将ds放到threadlocal中
```



3）谁要用ds

```
 DynamicRoutingDataSource. determineDataSource()  确定当前线程使用哪个数据源
     dataSourceMap.get(ds)  
```



4）dataSourceMap怎么来？

```
   DynamicRoutingDataSource. afterPropertiesSet()  extends InitialBean覆盖
        DynamicDataSourceProvider. loadDataSources()
             AbstractJdbcDataSourceProvider. loadDataSources() jdbc数据源
             YmlDynamicDataSourceProvider. loadDataSources() 从yaml文件中加载数据源
        DynamicDataSourceProvider. addDataSource()
```



##### 7.2 BeanFactoryTransactionAttributeSourceAdvisor*事务

1）切入点pointcut（拦截什么）

```
TransactionAttributeSourcePointcut
     TransactionAttributeSourceClassFilter.match()
          TransactionAttributeSource. isCandidateClass()
               TransactionAnnotationParser. isCandidateClass() 
                   AnnotationUtils.isCandidateClass(targetClass, Transactional.class) 
```

从spring jpa 的接口实现 类或方法要带有 @Transactional 注解



2）事务信息

```
TransactionDefinition 事务定义

     TransactionAttribute 事务属性

          TransactionAttributeSource

              AnnotationTransactionAttributeSource
```



3）通知 advice（做什么）

```
 TransactionInterceptor
     TransactionAspectSupport. invokeWithinTransaction()
          TransactionAspectSupport. invokeWithinTransaction() 
               TransactionAspectSupport.invokeWithinTransaction() 
                   定义协议 交给 各数据库层实现 
```



4）开始

AbstractSingletonProxyFactoryBean

```
 AbstractSingletonProxyFactoryBean. afterPropertiesSet()
     ProxyFactory. addAdvisor()
          AbstractSingletonProxyFactoryBean.preInterceptors 事务执行前置函数 
          TransactionProxyFactoryBean. createMainInterceptor()核心返回TransactionInterceptor
          AbstractSingletonProxyFactoryBean. postInterceptors 事务执行后置函数
```