@SpringBootApplication源码如下：

```
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Configuration
@EnableAutoConfiguration
@ComponentScan
public @interface SpringBootApplication {
    Class<?>[] exclude() default {};

    String[] excludeName() default {};

    @AliasFor(
        annotation = ComponentScan.class,
        attribute = "basePackages"
    )
    String[] scanBasePackages() default {};

    @AliasFor(
        annotation = ComponentScan.class,
        attribute = "basePackageClasses"
    )
    Class<?>[] scanBasePackageClasses() default {};
}
```



从源代码中得知 @SpringBootApplication 被 @Configuration、@EnableAutoConfiguration、@ComponentScan 注解所修饰，换言之 Springboot 提供了统一的注解来替代以上三个注解，简化程序的配置。下面解释一下各注解的功能。

#### 1.@Configuration

spring文档说明：

> @Configuration is a class-level annotation indicating that an object is a source of bean definitions. @Configuration classes declare beans via public @Bean annotated methods.
>

@Configuration 是一个类级注释，指示对象是一个bean定义的源。@Configuration 类通过 @bean 注解的公共方法声明bean。



> The @Bean annotation is used to indicate that a method instantiates, configures and initializes a new object to be managed by the Spring IoC container.
>

@Bean 注释是用来表示一个方法实例化，配置和初始化是由 Spring IoC 容器管理的一个新的对象。



通俗的讲 @Configuration 一般与 @Bean 注解配合使用，用 @Configuration 注解类等价与 XML 中配置 beans，用 @Bean 注解方法等价于 XML 中配置 bean。



#### 2. @ComponentScan

spring文档说明：

> Configures component scanning directives for use with @Configuration classes. Provides support parallel with Spring XML’s context:component-scan element.

为 @Configuration注解的类配置组件扫描指令。同时提供与 Spring XML’s context:component-scan 元素并行的支持。



> Either basePackageClasses() or basePackages() (or its alias value()) may be specified to define specific packages to scan. If specific packages are not defined, scanning will occur from the package of the class that declares this annotation.

无论是 basePackageClasses() 或是 basePackages() （或其 alias 值）都可以定义指定的包进行扫描。如果指定的包没有被定义，则将从声明该注解的类所在的包进行扫描。

> 注：所以SpringBoot的启动类最好是放在root package下，因为默认不指定basePackages。



> Note that the context:component-scan element has an annotation-config attribute; however, this annotation does not. This is because in almost all cases when using @ComponentScan, default annotation config processing (e.g. processing @Autowired and friends) is assumed. Furthermore, when using AnnotationConfigApplicationContext, annotation config processors are always registered, meaning that any attempt to disable them at the @ComponentScan level would be ignored.

注意，context:component-scan 元素有一个 annotation-config 属性，但是 @ComponentScan 没有。这是因为在使用 @ComponentScan 注解的几乎所有的情况下，默认的注解配置处理是假定的。此外，当使用 AnnotationConfigApplicationContext， 注解配置处理器总会被注册，意味着任何试图在 @ComponentScan 级别是扫描失效的行为都将被忽略。



通俗的讲，@ComponentScan 注解会自动扫描指定包下的全部标有 @Component注解 的类，并注册成bean，当然包括 @Component 下的子注解@Service、@Repository、@Controller。@ComponentScan 注解没有类似 context:exclude-filter、context:include-filte的属性。



#### 3. @EnableAutoConfiguration

spring文档说明：

> Enable auto-configuration of the spring Application Context, attempting to guess and configure beans that you are likely to need. Auto-configuration classes are usually applied based on your classpath and what beans you have defined.
>

启用 Spring 应用程序上下文的自动配置，试图猜测和配置您可能需要的bean。自动配置类通常采用基于你的 classpath 和已经定义的 beans 对象进行应用。



> The package of the class that is annotated with @EnableAutoConfiguration has specific significance and is often used as a ‘default’. For example, it will be used when scanning for@Entity classes. It is generally recommended that you place@EnableAutoConfiguration in a root package so that all sub-packages and classes can be searched.
>

被 @EnableAutoConfiguration 注解的类所在的包有特定的意义，并且作为默认配置使用。例如，当扫描 @Entity类的时候它将被使用。通常推荐将 @EnableAutoConfiguration 配置在 root 包下，这样所有的子包、类都可以被查找到。



> Auto-configuration classes are regular Spring Configuration beans. They are located using the SpringFactoriesLoader mechanism (keyed against this class).Generally auto-configuration beans are @Conditional beans (most often using @ConditionalOnClass and @ConditionalOnMissingBean annotations).
>

Auto-configuration类是常规的 Spring 配置 Bean。它们使用的是 SpringFactoriesLoader 机制（以 EnableAutoConfiguration 类路径为 key）。通常 auto-configuration beans 是 @Conditional beans（在大多数情况下配合 @ConditionalOnClass 和 @ConditionalOnMissingBean 注解进行使用）。



##### 3.1 什么是SpringFactoriesLoader 机制？

SpringFactoriesLoader为Spring工厂加载器，该对象提供了loadFactoryNames方法，入参为factoryClass和classLoader即需要传入工厂类名称和对应的类加载器，方法会根据指定的classLoader，加载该类加器搜索路径下的指定文件，即spring.factories文件，传入的工厂类为接口，而文件中对应的类则是接口的实现类，或最终作为实现类。

```
public abstract class SpringFactoriesLoader {
    //...
    public static <T> List<T> loadFactories(Class<T> factoryClass, ClassLoader classLoader) {
        ...
    }
    public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
        ....
    }
}
```



所以文件中一般为如下图这种一对多的类名集合，获取到这些实现类的类名后，loadFactoryNames方法返回类名集合，方法调用方得到这些集合后，再通过**反射**获取这些类的类对象、构造方法，最终生成实例。

![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2018/8/6/1650e0c6e28a0407~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)



下图有助于我们形象理解自动配置流程：

![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2018/8/6/1650e0e47481e59a~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)