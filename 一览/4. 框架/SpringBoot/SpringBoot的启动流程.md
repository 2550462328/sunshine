

springBoot启动大致需要完成以下步骤

1. 创建一个 合适的 ApplicationContext
2. 将命令行参数 注入到 spring properties 中
3. refresh applicationContext
4. 调用启动回调接口（ComandLineRunner）



在全局有如下配置

1）事件监听 SpringApplicationRunListeners

```
 starting 准备启动
 environmentPrepared 环境准备就绪
 contextPrepared 上下文准备就绪
 contextLoaded 上下文环境加载完毕
 started 启动完毕 
 running 正在运行
 failed 异常事件
```



2）异常报告 SpringBootExceptionReporter

```
     SpringApplication. handleRunFailure()
          SpringBootExceptionReporter. reportFailure()  异常报告
```



#### 1. 准备 Enviroment

```
SpringApplication. prepareEnvironment
    SpringApplication.getOrCreateEnvironment()  初始化环境
          WebApplicationType. deduceFromClasspath() 从上下文推理当前环境                         SERVLET -> StandardServletEnvironment
                  REACTIVE -> StandardReactiveWebEnvironment
                  DEFAULT -> StandardEnvironment
     SpringApplication. configureEnvironment()  配置环境
          configurePropertySources()  配置命令行参数 到 Enviroment中                                                 （SimpleCommandLinePropertySource）
          configureProfiles()  配置命令行 模板 到 Enviroment 中
     ConfigurationPropertySources.attach() 适配Enviroment 的MutablePropertySources 到                                         SpringConfigurationPropertySources
     SpringApplication. bindToSpringApplication() 将 enviroment 属性 bind到当前上下文
          Binder.bind()
```

- SpringApplication. configureIgnoreBeanInfo() 配置是否允许搜索自定义BeanInfo
- SpringApplication. printBanner() 打印启动日志中的Banner ~



答疑时间

1）**StandardEnvironment？**



StandardServletEnvironment：对应SERVLET模式（阻塞的）

StandardReactiveWebEnvironment：对应REACTIVE模式（非阻塞的）



2）**Binder.bind**

spring 2.0 中绑定属性的用法

Binder.get(environment).bind("spring.main", Bindable.ofInstance(this));



上面的代码意思是 将environment中 spring.main开头的相关属性 装配到 SpringApplication类中



如下是spring.main 的相关

![img](http://pcc.huitogo.club/77f118b3124f6e90bad4a2c9e37257b3)



底层实现有：

ValueObjectBinder. bind() 利用构造函数 进行传值

JavaBeanBinder.bind() 利用bean 的 getter / setter 进行传值



3）spring.beaninfo.ignore -> Introspector -> BeanInfo

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



#### 2. 准备 ApplicationContext

```
 SpringApplication. createApplicationContext() 创建ApplicationContext （webApplicationType）
     SERVLET -> AnnotationConfigServletWebServerApplicationContext

     REACTIVE –> AnnotationConfigReactiveWebServerApplicationContext
     DEFAULT –> AnnotationConfigApplicationContext
	 
	 SpringApplication. prepareContext() 
     SpringApplication. postProcessApplicationContext()
          beanNameGenerator 注入 beanNameGenerator
          resourceLoader 配置 ResourceLoader
          resourceLoader.getClassLoader() 配置 ClassLoader
          sharedInstance 添加 ApplicationConversionService
     SpringApplication. applyInitializers()  applicationcontext 初始化事件
          ApplicationContextInitializer. initialize
```

SpringApplication.load() 注入 source （这里将我们的SpringBootApplication注入进去，开启SpringBoot 欢乐时光）



#### 3. 更新 ApplicationContext

SpringApplication. refreshContext() AbstractApplicationContext. refresh() 更新 Spring 上下文 ConfigurableApplicationContext. registerShutdownHook() 设置 spring 关闭 钩子函数





#### 4. 回调

SpringApplication. afterRefresh() 自定义 未实现（备用）

```
SpringApplication. callRunners() 回调事件处理
     ApplicationRunner. run() 应用回调
     CommandLineRunner. run() 命令行事件回调
```



#### 5. 其他问题

1）**注入SpringBootApplication?**

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



2）**yaml是怎么解析的？**

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