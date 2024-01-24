#### 1. 工厂模式

比如Spring使用工厂模式通过 BeanFactory、ApplicationContext 创建 bean 对象。



两者对比：

1）**BeanFactory** ：延迟注入(使用到某个 bean 的时候才会注入),相比于ApplicationContext 来说会占用更少的内存，程序启动速度更快。

```
1. DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory(); 

2.  

3. XmlBeanDefinitionReader definitionReader = new XmlBeanDefinitionReader(beanFactory); 

4.  

5. ClassPathResource resource = new ClassPathResource("applicationContext.xml"); 

6. definitionReader.loadBeanDefinitions(resource); 

7.  

8. System.out.println(beanFactory.getBeanDefinition("user")); 
```



2）**ApplicationContext** ：容器启动的时候，不管你用没用到，一次性创建所有 bean 。BeanFactory 仅提供了最基本的依赖注入支持， ApplicationContext 扩展了 BeanFactory ,除了有BeanFactory的功能还有额外更多功能，所以一般开发人员使用 ApplicationContext会更多。

```
1. ApplicationContext context = new ClasspathXmlApplicationContext( "application.xml"); 

2.  

3. User= (User) context.getBean("user"); 
```



#### 2. 代理模式

比如Spring AOP 功能的实现

![img](http://pcc.huitogo.club/de3586c16218049d77889ee20577b85f)



#### 3. 单例模式

比如Spring 中的 Bean 默认都是单例的。



Spring 通过 ConcurrentHashMap 实现单例注册表的特殊方式实现单例模式。

Spring 实现单例的核心代码如下

```
1. // 通过 ConcurrentHashMap（线程安全） 实现单例注册表 

2. private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(64); 

3.  

4. public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) { 

5.     Assert.notNull(beanName, "'beanName' must not be null"); 

6.     synchronized (this.singletonObjects) { 

7.       // 检查缓存中是否存在实例  

8.       Object singletonObject = this.singletonObjects.get(beanName); 

9.       if (singletonObject == null) { 

10.         //...省略了很多代码 

11.         try { 

12.           singletonObject = singletonFactory.getObject(); 

13.         } 

14.         //...省略了很多代码 

15.         // 如果实例对象在不存在，我们注册到单例注册表中。 

16.         addSingleton(beanName, singletonObject); 

17.       } 

18.       return (singletonObject != NULL_OBJECT ? singletonObject : null); 

19.     } 

20.   } 

21.   //将对象添加到单例注册表 

22.   protected void addSingleton(String beanName, Object singletonObject) { 

23.       synchronized (this.singletonObjects) { 

24.         this.singletonObjects.put(beanName, (singletonObject != null ? singletonObject : NULL_OBJECT));  

26.       } 

27.     } 

28. } 
```



#### 4. 模板方法模式

比如Spring 中 jdbcTemplate、hibernateTemplate 等以 Template 结尾的对数据库操作的类，它们就使用到了模板模式。一般情况下，我们都是使用继承的方式来实现模板模式，但是 Spring 并没有使用这种方式，而是使用Callback 模式与模板方法模式配合，既达到了代码复用的效果，同时增加了灵活性。



#### 5. 装饰者模式

Spring 中配置 DataSource 的时候，DataSource 可能是不同的数据库和数据源。我们能否根据客户的需求在少修改原有类的代码下动态切换不同的数据源？这个时候就要用到装饰者模式。

Spring 中用到的包装器模式在类名上含有 Wrapper或者 Decorator。这些类基本上都是动态地给一个对象添加一些额外的职责。



#### 6. 观察者模式

比如Spring 事件驱动模型



**事件角色**

ApplicationEvent (org.springframework.context包下)充当事件的角色,这是一个抽象类，它继承了java.util.EventObject并实现了 java.io.Serializable接口。



Spring 中默认存在以下事件，他们都是对 ApplicationContextEvent 的实现(继承自ApplicationContextEvent)：

- ContextStartedEvent：ApplicationContext 启动后触发的事件;
- ContextStoppedEvent：ApplicationContext 停止后触发的事件;
- ContextRefreshedEvent：ApplicationContext 初始化或刷新完成后触发的事件;
- ContextClosedEvent：ApplicationContext 关闭后触发的事件。



![img](http://pcc.huitogo.club/54a46422b2e70344ef158c2341e7018b)



**事件监听者角色**

ApplicationListener 充当了事件监听者角色，它是一个接口，里面只定义了一个 onApplicationEvent（）方法来处理ApplicationEvent。



所以，**在 Spring中我们只要实现 ApplicationListener 接口的 onApplicationEvent() 方法即可完成监听事件**



ApplicationListener接口类源码如下

```
1. package org.springframework.context; 

2. import java.util.EventListener; 

3. @FunctionalInterface 

4. public interface ApplicationListener<E extends ApplicationEvent> extends EventListener { 

5.   void onApplicationEvent(E var1); 

6. } 
```



**事件发布者角色**

ApplicationEventPublisher 充当了事件的发布者，它也是一个接口。

```
1. @FunctionalInterface 

2. public interface ApplicationEventPublisher { 

3.   default void publishEvent(ApplicationEvent event) { 

4.     this.publishEvent((Object)event); 

5.   } 

6.  

7.   void publishEvent(Object var1); 

8. } 
```



ApplicationEventPublisher 接口的publishEvent（）这个方法在AbstractApplicationContext类中被实现，在这个方法的实现中发现事件真正是通过ApplicationEventMulticaster来广播出去的。



所以spring事件模型的观察者模式是被动接受监听事件（消息）的。



我们可以基于这些自定义监听事件，示例代码如下：

```
1. // 定义一个事件,继承自ApplicationEvent并且写相应的构造函数 

2. public class DemoEvent extends ApplicationEvent{ 

3.   private static final long serialVersionUID = 1L; 

4.  

5.   private String message; 

6.  

7.   public DemoEvent(Object source,String message){ 

8.     super(source); 

9.     this.message = message; 

10.   } 

11.  

12.   public String getMessage() { 

13.     return message; 

14.      } 

15.   

17. // 定义一个事件监听者,实现ApplicationListener接口，重写 onApplicationEvent() 方法； 

18. @Component 

19. public class DemoListener implements ApplicationListener<DemoEvent>{ 

20.  

21.   //使用onApplicationEvent接收消息 

22.   @Override 

23.   public void onApplicationEvent(DemoEvent event) { 

24.     String msg = event.getMessage(); 

25.     System.out.println("接收到的信息是："+msg); 

26.   } 

27.  

28. } 

29. // 发布事件，可以通过ApplicationEventPublisher 的 publishEvent() 方法发布消息。 

30. @Component 

31. public class DemoPublisher { 

32.  

33.   @Autowired 

34.   ApplicationContext applicationContext; 

35.  

36.   public void publish(String message){ 

37.     //发布事件 

38.     applicationContext.publishEvent(new DemoEvent(this, message)); 

39.   } 

40. } 
```



#### 7. 适配器模式

**Spring AOP 的增强或通知(Advice)使用到了适配器模式**

Spring AOP 的增强或通知(Advice)使用到了适配器模式，与之相关的接口是AdvisorAdapter 。Advice 常用的类型有：BeforeAdvice（目标方法调用前,前置通知）、AfterAdvice（目标方法调用后,后置通知）、AfterReturningAdvice(目标方法执行结束后，return之前)等等。每个类型Advice（通知）都有对应的拦截器:MethodBeforeAdviceInterceptor、AfterReturningAdviceAdapter、AfterReturningAdviceInterceptor。Spring预定义的通知要通过对应的适配器，适配成 MethodInterceptor接口(方法拦截器)类型的对象（如：MethodBeforeAdviceInterceptor 负责适配 MethodBeforeAdvice）。



**spring MVC 中也是用到了适配器模式适配Controller**

在Spring MVC中，DispatcherServlet 根据请求信息调用 HandlerMapping，解析请求对应的 Handler。解析到对应的 Handler（也就是我们平常说的 Controller 控制器）后，开始由HandlerAdapter 适配器处理。HandlerAdapter 作为期望接口，具体的适配器实现类用于对目标类进行适配，Controller 作为需要适配的类。



**为什么要在 Spring MVC 中使用适配器模式？**

Spring MVC 中的 Controller 种类众多，不同类型的 Controller 通过不同的方法来对请求进行处理。如果不利用适配器模式的话，DispatcherServlet 直接获取对应类型的 Controller，需要的自行来判断，像下面这段代码一样：

```
1. if(mappedHandler.getHandler() instanceof MultiActionController){  

2.  ((MultiActionController)mappedHandler.getHandler()).xxx  

3. }else if(mappedHandler.getHandler() instanceof XXX){  

4.   ...  

5. }else if(...){  

6.  ...  

7. }  
```

假如我们再增加一个 Controller类型就要在上面代码中再加入一行 判断语句，这种形式就使得程序难以维护，也违反了设计模式中的开闭原则 – 对扩展开放，对修改关闭。