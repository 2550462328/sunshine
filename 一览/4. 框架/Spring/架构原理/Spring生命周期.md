关于Spring生命周期，网上的流程图千奇百怪，比如这样的

![img](http://pcc.huitogo.club/b0148eae30ec83be7a84efb5e2ba0a7c)



下面是我自己总结的一张图，可以对比着看

![img](http://pcc.huitogo.club/9a2b1a2a32a69ca2b4cc022acafbef8f)



#### 1. 加载相关构造器

这个部分没什么说的，在做这件事之前肯定要将主要涉及的类实例化起来，比如InstantiationAwareBeanPostProcessor、BeanPostProcessor、BeanFactoryPostProcessor这些类和子类（我们扩展的类）



值得一提的就是在实例化这些相关类后，会执行BeanFactoryPostProcessor.postProcessBeanFactory()这个方法，用来在正式装配Bean之前对装配工厂BeanFactory进行一些设置，也可以通过FactoryBean提前注入我们需要的类。



#### 2. 实例化

实例化过程类似于我们Java中的new Object()



但是在Spring框架中这个过程是通过反射实现的

在实例化中主要影响相关的就是InstantiationAwareBeanPostProcessor类

- **实例化前**：InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation()
- **实例化**：使用反射生成，如果Bean是通过构造器注入的话会使用带参构造方法反射生成
- **实例化后**：InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation()



#### 3. 填充属性

比如说典型的xml中配置

```
1. <bean id = "personService" class="cn.zhanghui.demo.daily.my_spring.beanfactory_set.test.service.PersonService"> 

2.   <property name="eatDao" ref="eatDao" /> 

3.   <property name="drinkDao" ref="drinkDao" /> 

4.   <property name="name" value="zhanghui" /> 

5.   <property name="age" value="18" /> 

6. </bean> 
```



这些property就会在这个阶段被设置到上个阶段实例化后的Bean中



在设置属性过程中主要的方法就是

- **设置属性前**：InstantiationAwareBeanPostProcessor.postProcessPropertyValues()
- **设置属性时**如果是按上述xml配置的话，会调用Bean的set方法为它设置属性。



除了set设置属性外，还有构造器实例化时设置属性和注解设置属性。

当然我们可以通过一些接口获取Spring容器的属性

比如

- BeanNameAware.setBeanName：获取和设置beanName属性
- BeanFactoryAware.setBeanFactory：获取和设置beanFactory属性



#### 4. 初始化

现在Bean已经实例化了，里面属性已经设置好了，已经是一个成熟的Bean了，但是我们还是希望它在正式使用前能再包装一下，也就是初始化过程。



初始化相关的类典型代表就是BeanPostProcessor

- **全局Bean初始化之前**：BeanPostProcessor.postProcessBeforeInitialization()
- **不常用的单个Bean初始化之前**：InitializingBean.afterPropertiesSet()
- **常用的单个Bean初始化前**：

在xml中配置

```
1. <bean id="user" class="cn.zhanghui.demo.daily.spring.lifestyle.User" init-method="myInit" /> 
```

用注解的话就是@PostConstruct

- **全局Bean初始化之后**：BeanPostProcessor.postProcessAfterInitialization()



#### 5. 使用Bean

初始化后的Bean就可以正式被我们使用了，比如ApplicationContext.getBean()

还有常见的@Autowired、@Resource注入的Bean

当然这个Bean是在它的scope范围内使用的，常见的是singleton，只要Spring Container不关闭，我就一直在



#### 6. 销毁

Bean走到了scope尽头了，或者容器关闭了，那么Bean应该被正确的销毁。

销毁针对单个Bean

- **不常用的单个Bean销毁之前**：DisposableBean.destroy()
- **常用的单个Bean销毁之前**：

在xml中配置

```
1. <bean id="user" **class**="cn.zhanghui.demo.daily.spring.lifestyle.User" destroy-method="myDestory" /> 
```

用注解的话就是@PreDestroy



#### 7. 总结

![img](http://pcc.huitogo.club/b2d197f9ebc2bc749f59d481729e8b52)



- **Bean 自身的方法**：如调用 Bean 构造函数实例化 Bean，调用 Setter 设置 Bean 的属性值以及通过的 init-method 和 destroy-method 所指定的方法；

- **Bean 级生命周期接口方法**：如 BeanNameAware、 BeanFactoryAware、 InitializingBean 和 DisposableBean，这些接口方法由 Bean 类直接实现；

- **容器级生命周期接口方法**：在上图中带“★” 的步骤是由 InstantiationAwareBean PostProcessor 和 BeanPostProcessor 这两个接口实现，一般称它们的实现类为“ 后处理器” 。 后处理器接口一般不由 Bean 本身实现，它们独立于 Bean，实现类以容器附加装置的形式注册到 Spring 容器中并通过接口反射为 Spring 容器预先识别。当Spring 容器创建任何 Bean 的时候，这些后处理器都会发生作用，所以这些后处理器的影响是全局性的。当然，用户可以通过合理地编写后处理器，让其仅对感兴趣Bean 进行加工处理。



**总结**过来就是

Spring真的是一个扩展性极高的容器，从本章Bean生命周期的过程就可以看的出来，我们几乎可以在Bean的任意时期做一些什么事情，当然基本开发的话是不会考虑这些过程，主要还是对框架研发的时候。