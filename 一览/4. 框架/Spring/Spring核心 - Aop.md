#### 1. AOP使用

1. 创建Advisor，用注解@Aspect标注，用@Pointcut指定切点，再用@Before、@After、@Test等注解标注方法在切点附近的执行时机
2. 在配置文件中声明aop:aspectj-autoproxy标签，并添加bean和测试类
3. 执行切点方法，查看Advisor中的方法是否执行
   



接下来从源码角度解读Aop

#### 2. 开始

![img](http://pcc.huitogo.club/9472548801489f5357f172fc5453fd2a)



##### 2.1 注册aop标签解析器

AopNamespaceHandler 对 `<aop:/>` 命名空间的处理器，主要是注册AspectJAutoProxyBeanDefinitionParser解析器

```
public void init() {
		// In 2.0 XSD as well as in 2.1 XSD.
		registerBeanDefinitionParser("config", new ConfigBeanDefinitionParser());
		registerBeanDefinitionParser("aspectj-autoproxy", new AspectJAutoProxyBeanDefinitionParser());
		registerBeanDefinitionDecorator("scoped-proxy", new ScopedProxyBeanDefinitionDecorator());

		// Only in 2.0 XSD: moved to context namespace as of 2.1
		registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
	}
```

- AspectJAutoProxyBeanDefinitionParser 
- ScopedProxyBeanDefinitionDecorator
- ConfigBeanDefinitionParser
- SpringConfiguredBeanDefinitionParser



##### 2.2 注册ProxyCreator

AspectJAutoProxyBeanDefinitionParser 的parse方法会注册AbstractAutoProxyCreator

核心代码在AopNamespaceUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary中：

1. 注册或升级AbstractAutoProxyCreator，它可以根据@Point注解去定义切点，自动代理相匹配的bean。

2. 调用useClassProxyingIfNecessary方法，处理proxy-target-class以及expose-proxy属性。

   

注意：这里AbstractAutoProxyCreator是个抽象类，具体是环境中注册的是它的实现类

- 在<aop->配置环境中注册的是AnnotationAwareAspectJAutoProxyCreator
- 在SpringBoot启动下注册的是InfrastructureAdvisorAutoProxyCreator



#### 3. 创建AOP代理

AbstractAutoProxyCreator实现了BeanPostProcessor接口，当这个bean被加载的时候，会自动调用postProcessAfterInitialization方法。在该方法中，会根据bean是否需要被代理，决定是否需要封装指定的bean，该操作在wrapIfNecessary方法中执行。

1. 判断bean是否已经处理过，如果在targetSourcedBeans中包含这个bean的名字，则说明已处理过，直接return
2. 判断bean是否需要处理，如果nonAdvisedBeans中包含这个bean的名字，则说明无需处理，直接return
3. 判断bean是否需要设置代理，如果该bean代表一个基础设施类，则不应设置代理，如果在配置中指定了这个bean不需要代理，则将其添加到nonAdvisedBeans中并直接return
4. 尝试通过getAdvicesAndAdvisorsForBean方法获取当前bean的增强方法，如果获取到了增强方法，则针对增强创建代理。创建代理的过程在createProxy方法中完成

![img](http://pcc.huitogo.club/86b2c42fed20357de0ff9d5132ec60b7)



##### 3.1 获取Adivsor

获取增强器的功能在findEligibleAdvisors方法中完成，通过注解实现AOP时，这个方法的实现是由AnnotationAwareAspectJAutoProxyCretor完成的，该方法间接继承了AbstractAdvisorAutoProxyCreator，在实现获取增强的方法中除了保留父类的获取配置文件中定义的增强外，同时添加了获取bean的注解增强的功能。其大致过程包括：获取所有的beanName、遍历所有beanName，找出声明AspectJ注解的类、对标记为AspectJ注解的类进行增强器提取(getAdvisors)，最后将提取结果存入缓存中。



- advisorFactory.getAdvisors

增强器的提取在getAdvisors方法中完成，具体有两个步骤：获取切点的注解以及根据注解信息生成增强

​				a）切点信息获取：获取指定注解的表达式信息，如注解类型、注解内的表达式等

​		        b）根据切点信息，调用Advisor类的InstantiationModelAwarePointcutAdvisorImpl方法，根据不同的注解类型封装不同的增强器

- SyntheticInstantiationAdvisor

如果寻找的增强器不为空，而且又配置了增强的延迟初始化，那么需要在首位加入同步实例化增强器

- getDeclareParentsAdvisor

用于获取DeclareParents注解，DeclareParents主要用于引介增强的注解形式的实现，而其实现方式与普通增强类似，但会使用DeclareParentsAdvisor对功能进行封装



##### 3.2 过滤Advisor

针对于每个bean，要挑选出适合当前bean的增强器，即满足我们配置的通配符的增强器，该过程在AopUtil.findAdvisorsThatCanApply方法中实现，其主要功能是寻找所有适用于当前bean的增强器，并分开处理引介增强和普通增强。对于是否适用。



##### 3.3 Advisor排序

AnnotationAwareOrderComparator



***Q1：为什么ExposeInvocationInterceptor放在所有MethodInterceptor的前面？***

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



##### 3.3 创建代理

createProxy过程：

1. 获取当前类中的相关属性
2. 添加代理接口
3. 封装Advisor并加入ProxyFactory中
4. 设置要代理的类
5. 根据需要定制代理类
6. 进行获取代理的操作，调用proxyFactroy的getProxy方法获取代理



getProxy：

```
public Object getProxy(ClassLoader classLoader){
    return createAopProxy().getProxy(classLoader);
}
```



创建代理核心方法：AopProxyFactory.createAopProxy

主要是根据一些条件来确定创建JDKProxy还是CglibProxy。根据if条件，有三个方面的因素：

1. optimize：用来控制通过CGLIB创建的代理是否使用激进优化策略，只推荐在完全了解AOP代理的优化的情况下使用。对JDK动态代理无效。

2. proxyTargetClass：这个属性为true时，目标本身被代理而不是目标类的接口，此时CGLIB代理将被创建。设置方式:

   ```
   <aop:aspextj-autoproxy proxy-target-class = “true” />
   ```

3. hasNoUserSuppliedProxyInterfaces：是否存在代理接口

   

JDK与Cglib方式的总结：

1. 如果目标对象实现了接口，默认情况下采用JDK的动态代理实现AOP
2. 如果目标对象实现了接口，可采用添加CGLIB库或指定proxy-target-class的方法强制使用CGLIB实现AOP
3. 如果没有实现接口，则必须采用CGLIB库，Spring会自动在两者之间转换



两者的区别：

- JDK动态代理只能对实现了接口的类生成代理，而不能针对类
- CGLIB时针对类实现代理，主要是对指定的类生成子类，覆盖其中的方法。因为这个实现过程本质上是继承，所以类或方法最好不声明为final的
  

#### 4. 链式执行

Advice -> 对应 MethodInterceptor 怎么执行 Advisor -> Advice 的适配器 什么时候执行（切入点） + 怎么执行 Advised -> 代理类容器 = Interceptors + other advice + Advisors + the proxied interfaces

AdvisedSupport.getInterceptorsAndDynamicInterceptionAdvice 获取代理方法上 真正执行的 拦截器链 （一堆Advisor）

AdvisorAdapter. getInterceptor(Advisor advisor) 转换Advisor 到对应的 MethodInteceptor AdvisorAdapterRegistry. registerAdvisorAdapter 注册AdvisorAdapter （so 可以自定义）

![img](http://pcc.huitogo.club/015795c232e14e34a0b92805cc9197cf)



最后执行的时候是执行 Advisor 所对应的 MethodInteceptor

BeforeAdvice、AfterAdvice、AbstractAspectJAdvice

![img](http://pcc.huitogo.club/843e5a56e8a4bf51cbb54107aa926c9c)



#### 5. 扩展

##### 5.1 DynamicDataSourceAnnotationAdvisor 动态数据源 

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



##### 5.2 BeanFactoryTransactionAttributeSourceAdvisor*事务

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