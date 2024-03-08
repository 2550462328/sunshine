springBoot启动大致需要完成以下步骤

1. 准备环境
2. 创建一个 合适的 ApplicationContext
3. 将命令行参数 注入到 spring properties 中
4. Refresh applicationContext
5. 调用启动回调接口（ComandLineRunner）

其中穿插了很多监听器的动作，并且很多逻辑都是靠各种监听器的实现类执行的。



SpringBootApplication.run源码：

```
/**
 * Run the Spring application, creating and refreshing a new
 * {@link ApplicationContext}.
 * @param args the application arguments (usually passed from a Java main method)
 * @return a running {@link ApplicationContext}
 */
public ConfigurableApplicationContext run(String... args) {
   //开启时钟计时
   StopWatch stopWatch = new StopWatch();
   stopWatch.start();
   //spirng 上下文
   ConfigurableApplicationContext context = null;
   //启动异常报告容器
   Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
   //开启设置，让系统模拟不存在io设备
   configureHeadlessProperty();
   // 1 初始化SpringApplicationRunListener 监听器，并进行封装
   SpringApplicationRunListeners listeners = getRunListeners(args);
   listeners.starting();
   try {
      ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
     //2 Environment 的准备 
      ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
      configureIgnoreBeanInfo(environment);
      Banner printedBanner = printBanner(environment); // 打印标语 彩蛋
     //3 创建上下文实例
      context = createApplicationContext();
     //异常播报器，默认有org.springframework.boot.diagnostics.FailureAnalyzers
      exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
            new Class[] { ConfigurableApplicationContext.class }, context);
     //4 容器初始化
      prepareContext(context, environment, listeners, applicationArguments, printedBanner);
     //5 刷新上下文容器 
      refreshContext(context);
     //给实现类留的钩子，这里是一个空方法。
      afterRefresh(context, applicationArguments);
      stopWatch.stop();
      if (this.logStartupInfo) {
         new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
      }
      listeners.started(context);
      callRunners(context, applicationArguments);
   }
   catch (Throwable ex) {
      handleRunFailure(context, ex, exceptionReporters, listeners);
      throw new IllegalStateException(ex);
   }

   try {
      listeners.running(context);
   }
   catch (Throwable ex) {
      handleRunFailure(context, ex, exceptionReporters, null);
      throw new IllegalStateException(ex);
   }
   return context;
}
```



SpringBoot启动流程图：

![spring_running.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6ba8bf5c8177430b8f462f35948d1c74~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)



#### 1. SpringApplicationRunListener使用

首先通过getSpringFactoriesInstances 获取到所有实现SpringApplicationRunListener  接口的实例，默认情况下该接口的实现类只有 EventPublishingRunListener  他的主要作用是作为Springboot 的一个广播器

```
public interface SpringApplicationRunListener {
  /**EventPublishingRunListener 前期采用 SimpleApplicationEventMulticaster.multicastEvent(ApplicationEvent) 进行广播
  **/
   default void starting() {} 
   default void environmentPrepared(ConfigurableEnvironment environment) {}
   default void contextPrepared(ConfigurableApplicationContext context) {}
   default void contextLoaded(ConfigurableApplicationContext context) {}
  /**
  EventPublishingRunListener 后期采用 context.publishEvent(ApplicationEvent)
  **/
   default void started(ConfigurableApplicationContext context) {}
   default void running(ConfigurableApplicationContext context) {}
   default void failed(ConfigurableApplicationContext context, Throwable exception) {}
}
```

-  starting 准备启动
-  environmentPrepared 环境准备就绪
-  contextPrepared 上下文准备就绪
-  contextLoaded 上下文环境加载完毕
-  started 启动完毕 
-  running 正在运行
-  failed 异常事件



#### 2. 准备 Enviroment

```
SpringApplication. prepareEnvironment
    SpringApplication.getOrCreateEnvironment()  初始化环境
          WebApplicationType. deduceFromClasspath() 从上下文推理当前环境                                       SERVLET -> StandardServletEnvironment
                  REACTIVE -> StandardReactiveWebEnvironment
                  DEFAULT -> StandardEnvironment
     SpringApplication. configureEnvironment()  配置环境
          configurePropertySources()  配置命令行参数 到 Enviroment中                                                 （SimpleCommandLinePropertySource）
          configureProfiles()  配置命令行模板到Enviroment 中
     ConfigurationPropertySources.attach() 适配Enviroment 的MutablePropertySources 到                                         SpringConfigurationPropertySources
     SpringApplication. bindToSpringApplication() 将 enviroment 属性 bind到当前上下文
          Binder.bind()
```

- SpringApplication. configureIgnoreBeanInfo() 配置是否允许搜索自定义BeanInfo
- SpringApplication. printBanner() 打印启动日志中的Banner ~



答疑时间

***Q1：StandardEnvironment？***

- StandardServletEnvironment：对应SERVLET模式（阻塞的）

- StandardReactiveWebEnvironment：对应REACTIVE模式（非阻塞的）




***Q2：Binder.bind***？

spring 2.0 中绑定属性的用法

```
Binder.get(environment).bind("spring.main", Bindable.ofInstance(this));
```

上面的代码意思是 将environment中 spring.main开头的相关属性 装配到 SpringApplication类中



如下是spring.main 的相关

![img](http://pcc.huitogo.club/77f118b3124f6e90bad4a2c9e37257b3)



底层实现有：

ValueObjectBinder. bind() 利用构造函数 进行传值

JavaBeanBinder.bind() 利用bean 的 getter / setter 进行传值



**Q3：spring.beaninfo.ignore -> Introspector -> BeanInfo？**

Introspector 是 javaBean 的内省工具（内省会按照类继承关系一层层向上内省）

```
1.    // 在Object类时候停止检索，可以选择在任意一个父类停止 

2.    BeanInfo beanInfo = Introspector.getBeanInfo(JavaBeanDemo.class,Object.class); 
```



getBeanInfo流程

![img](http://pcc.huitogo.club/da87413fd178a7d30347e014ed7bc111)



BeanInfo只是一个内省结果的接口，Java中对该接口的实现有以下四种：

-  ApplicationBeanInfo：Apple desktop相关的JavaBean内省结果
-  ComponentBeanInfo：Java Awt组件的内省结果，如按钮等
-  GenericBeanInfo：通用的内省结果，JEE开发中的内省结果都为该类型
-  ExtendedBeanInfo：Spring自定义的内省结果类型，主要用于识别返回值不为空的Setter方法



spring.beaninfo.ignore = true 表示 禁止对自定义的BeanInfo 类的搜索（除上述BeanInfo接口的实现类之外的BeanInfo实现）

Spring的BeanUtils.copyProperties底层就是使用的javaBean的内省，通过内省得到拷贝源对象和目的对象属性的读方法和写方法，然后调用对应的方法进行属性的复制

![img](http://pcc.huitogo.club/1e7e34ca56d6acbcd7bcfbc02aade958)



#### 3. 创建 ApplicationContext

```
 SpringApplication. createApplicationContext() 创建ApplicationContext （webApplicationType）
     SERVLET -> AnnotationConfigServletWebServerApplicationContext
     REACTIVE –> AnnotationConfigReactiveWebServerApplicationContext
     DEFAULT –> AnnotationConfigApplicationContext
	 
	 SpringApplication. prepareContext() 
         SpringApplication. postProcessApplicationContext()
              beanNameGenerator 注入 beanNameGenerator
              resourceLoader  配置 ResourceLoader
              resourceLoader.getClassLoader()  配置 ClassLoader
              sharedInstance 添加 ApplicationConversionService
         SpringApplication. applyInitializers()  applicationcontext 初始化事件
              ApplicationContextInitializer. initialize
```

- SpringApplication.load() 注入 source （这里将我们的SpringBootApplication注入进去，开启SpringBoot 欢乐时光）




其中`#prepareContext()`初始化ApplicationContext上下文

```
private void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment,
      SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
  //绑定环境
   context.setEnvironment(environment);
  //如果application有设置beanNameGenerator、resourceLoader就将其注入到上下文中，并将转换工具也注入到上下文中
  postProcessApplicationContext(context);
  //调用初始化的切面
   applyInitializers(context);
  //发布ApplicationContextInitializedEvent事件
   listeners.contextPrepared(context);
  //日志
   if (this.logStartupInfo) {
      logStartupInfo(context.getParent() == null);
      logStartupProfileInfo(context);
   }
   // Add boot specific singleton beans
   ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
   beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
   if (printedBanner != null) {
      beanFactory.registerSingleton("springBootBanner", printedBanner);
   }
   if (beanFactory instanceof DefaultListableBeanFactory) {
     //如果bean名相同的话是否允许覆盖，默认为false，相同会抛出异常
      ((DefaultListableBeanFactory) beanFactory)
            .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
   }
   if (this.lazyInitialization) {
      context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());
   }
   // Load the sources
  // 这里获取到的是BootstrapImportSelectorConfiguration这个class，而不是自己写的启动来，这个class是在之前注册的BootstrapApplicationListener的监听方法中注入的
   Set<Object> sources = getAllSources();
   Assert.notEmpty(sources, "Sources must not be empty");
  //加载sources 到上下文中
   load(context, sources.toArray(new Object[0]));
  //发布ApplicationPreparedEvent事件
   listeners.contextLoaded(context);
}
```



#### 4. 刷新 ApplicationContext

```
SpringApplication. refreshContext() 

  AbstractApplicationContext. refresh() 更新 Spring 上下文 

  ConfigurableApplicationContext. registerShutdownHook() 设置 spring 关闭 钩子函数
```



其中`#AbstractApplicationContext.refresh`的核心可以看Spring的refesh源码

```
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        //记录启动时间、状态，web容器初始化其property，复制listener
        prepareRefresh();
        //这里返回的是context的BeanFactory
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
        //beanFactory注入一些标准组件，例如ApplicationContextAwareProcessor，ClassLoader等
        prepareBeanFactory(beanFactory);
        try {
            //给实现类留的一个钩子，例如注入BeanPostProcessors，这里是个空方法
            postProcessBeanFactory(beanFactory);

            // 调用切面方法
            invokeBeanFactoryPostProcessors(beanFactory);

            // 注册切面bean
            registerBeanPostProcessors(beanFactory);

            // Initialize message source for this context.
            initMessageSource();

            // bean工厂注册一个key为applicationEventMulticaster的广播器
            initApplicationEventMulticaster();

            // 给实现类留的一钩子，可以执行其他refresh的工作，这里是个空方法
            onRefresh();

            // 将listener注册到广播器中
            registerListeners();

            // 实例化未实例化的bean
            finishBeanFactoryInitialization(beanFactory);

            // 清理缓存，注入DefaultLifecycleProcessor，发布ContextRefreshedEvent
            finishRefresh();
        }

        catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - " +
                        "cancelling refresh attempt: " + ex);
            }

            // Destroy already created singletons to avoid dangling resources.
            destroyBeans();

            // Reset 'active' flag.
            cancelRefresh(ex);

            // Propagate exception to caller.
            throw ex;
        }

        finally {
            // Reset common introspection caches in Spring's core, since we
            // might not ever need metadata for singleton beans anymore...
            resetCommonCaches();
        }
    }
}
```



#### 5. 回调

SpringApplication. afterRefresh() 自定义 未实现（备用）

```
SpringApplication. callRunners() 回调事件处理
     ApplicationRunner. run() 应用回调
     CommandLineRunner. run() 命令行事件回调
```



#### 6. 其他问题

***Q1：SpringBootApplication自动配置?***

```
@SpringBootConfiguration  =  @Configuration 不过在SpringBoot项目中只能有一个
 
 @EnableAutoConfiguration 自动装配springBoot 上下文*
     @AutoConfigurationPackage 
          // 将BasePackage（有启动类目录信息） 作为bean装载到spring中
          AutoConfigurationPackages. register

     // 获取bean 用于装配到spring中
     AutoConfigurationImportSelector. selectImports()  
          AutoConfigurationImportSelector. getAutoConfigurationEntry() 
               AutoConfigurationImportSelector. getCandidateConfigurations()
                   SpringFactoriesLoader.loadFactoryNames()                           //从spring环境中的加载META-INF/spring.factories，获取EnableAutoConfiguration下的类
                        SpringFactoriesLoader. loadSpringFactories()
```

![img](http://pcc.huitogo.club/8140542aa852198755f97b104f0e99e5)



@EnableAutoConfiguration 适合 和 @Conditional 在一起食用

对于自动装配的bean 如果缺少必要的依赖 和 属性值则没有装配进spring的必要



当然 加载完之后会有去重、排除项（在@SpringBootApplication中配置）、过滤（AutoConfigurationImportFilter），最后是调用自动装配完成的监听器（AutoConfigurationImportListener）



**BasePackage的 bean什么时候被用？**

其他功能组件 需要扫描 当前上下文Bean 的时候，比如jpa扫描Entiry

```
 EntityScanner.scan()
     EntityScanner. getPackages()
          AutoConfigurationPackages. getBean(BEAN, BasePackages.class)
```



***Q2：yaml是怎么解析的？***

**第一种被绑定到ConfigurationProperties上（@ConfigurationProperties）**

入口

```
@EnableConfigurationProperties 
     // 将@ConfigurationProperties 注解的类装配到spring中 
     EnableConfigurationPropertiesRegistrar. registerBeanDefinitions() 

     // 注入绑定ConfigurationProperties的 处理器
    ConfigurationPropertiesBindingPostProcessor.register()
```



处理器

```
 ConfigurationPropertiesBindingPostProcessor. postProcessBeforeInitialization()
     // 核心绑定逻辑
     ConfigurationPropertiesBinder.bind()
          // 基于 bean 绑定 或者 构造器绑定 这里对于属性绑定肯定是bean
          Binder.bind() 
```



**第二种被SPEL表达式解析给属性赋值（@Value）**

入口

```
PropertyPlaceholderAutoConfiguration
// 加载environmentProperties 和 localProperties
 PropertySourcesPlaceholderConfigurer. postProcessBeanFactory()
     // 解析 SPEL 表达式
     PropertySourcesPlaceholderConfigurer. processProperties()
          // 对每个 BeanDefinition 的 属性进行解析
          BeanDefinitionVisitor. visitBeanDefinition()
               ConfigurablePropertyResolver. resolvePlaceholders()
                   // 将SPEL表达式替换成实际值的核心方法
                   PropertyPlaceholderHelper. parseStringValue() 
```