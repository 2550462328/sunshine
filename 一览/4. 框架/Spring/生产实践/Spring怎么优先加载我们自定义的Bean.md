**在spring中怎么让我们自定义的优先级最高，优先于其他bean加载？**



尝试一些方案

1. @Order 注解或者实现 org.springframework.core.Ordered

   不可行。Order 相关的方法一般用来控制 Spring 自身组件相关 Bean 的顺序，比如 ApplicationListener，RegistrationBean 等，对于我们自己使用 @Service @Compont 注解注册的业务相关的 bean 没有排序的效果。

2. @AutoConfigureOrder/@AutoConfigureAfter/@AutoConfigureBefore 注解

   不可行。它们和 Ordered 一样都是针对 Spring 自身组件 Bean 的顺序。

3. @DependsOn 注解

   可行但麻烦。需要让每个每个依赖 SystemConfigService的 Bean 都改代码加上注解。



更简化的方案思路：自定义BeanDefinitionRegistryPostProcessor 注册我们自定义bean



首先了解一下ConfigurationClassPostProcessor、BeanFactoryPostProcessor、BeanDefinitionRegistryPostProcessor 和 BeanPostProcessor的加载顺序

**1）ConfigurationClassPostProcessor**

ConfigurationClassPostProcessor，实现了 BeanDefinitionRegistryPostProcessor 接口。它是一个非常非常重要的类，甚至可以说它是 Spring boot 提供的扫描你的注解并解析成 BeanDefinition 最重要的组件。我们在使用 SpringBoot 过程中用到的 @Configuration、@ComponentScan、@Import、@Bean 这些注解的功能都是通过 ConfigurationClassPostProcessor 注解实现的。



**2）BeanFactoryPostProcessor**

在 BeanFactory 初始化之后调用，来定制和修改 BeanFactory 的内容

所有的 Bean 定义（BeanDefinition）已经保存加载到 beanFactory，但是 Bean 的实例还未创建

方法的入参是 ConfigurrableListableBeanFactory，意思是你可以调整 ConfigurrableListableBeanFactory 的配置



**3）BeanDefinitionRegistryPostProcessor**

是 BeanFactoryPostProcessor 的子接口

在所有 Bean 定义（BeanDefinition）信息将要被加载，Bean 实例还未创建的时候加载

优先于 BeanFactoryPostProcessor 执行，利用 BeanDefinitionRegistryPostProcessor 可以给 Spring 容器中自定义添加 Bean

方法入参是 BeanDefinitionRegistry，意思是你可以调整 BeanDefinitionRegistry 的配置



**4）BeanPostProcessor**

在 Bean 实例化之后执行的

执行顺序在 BeanFactoryPostProcessor 之后

方法入参是 Object bean，意思是你可以调整 bean 的配置



最终实现思路：

##### 1. 注册BeanDefinitionRegistryPostProcessor

通过 spring.factories 扩展来注册一个 ApplicationContextInitializer

```
# 注册 ApplicationContextInitializer
org.springframework.context.ApplicationContextInitializer=com.antbank.demo.bootstrap.MyApplicationContextInitializer
```



注册BeanDefinitionRegistryPostProcessor 到 Spring 中

```
public class MyApplicationContextInitializer implements ApplicationContextInitializer<ConfigurableApplicationContext> {
    
    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        // 注意，如果你同时还使用了 spring cloud，这里需要做个判断，要不要在 spring cloud applicationContext 中做这个事
        // 通常 spring cloud 中的 bean 都和业务没关系，是需要跳过的
        applicationContext.addBeanFactoryPostProcessor(new MyBeanDefinitionRegistryPostProcessor());
    }
}
```



除了使用 spring 提供的 SPI 来注册 ApplicationContextInitializer，你也可以用 SpringApplication.addInitializers 的方式直接在 main 方法中直接注册一个 ApplicationContextInitializer 结果都是可以。

当然了，通过 Spring 的事件机制也可以做到注册 BeanDefinitionRegistryPostProcessor，选择实现合适的 ApplicationListener 事件，可以通过 ApplicationContextEvent 获得 ApplicationContext，即可注册 BeanDefinitionRegistryPostProcessor。



##### 2. 注册自定义bean

```
public class MyBeanDefinitionRegistryPostProcessor implements BeanDefinitionRegistryPostProcessor {
    
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
        // 手动注册一个 BeanDefinition
        registry.registerBeanDefinition("systemConfigService", new RootBeanDefinition(SystemConfigService.class));
    }
    
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {}
}
```



当然你也可以使用一个类同时实现 ApplicationContextInitializer 和BeanDefinitionRegistryPostProcessor。

通过 applicationContext#addBeanFactoryPostProcessor 注册的 BeanDefinitionRegistryPostProcessor，比 Spring 自带的优先级要高，所以这里就不需要再实现 Ordered 接口提升优先级就可以排在 ConfigurationClassPostProcessor 前面