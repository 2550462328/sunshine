**什么是@Conditional？**

@Conditional是由Spring 4提供的一个新特性，用于根据特定条件来控制Bean的创建行为。而在我们开发基于Spring的应用的时候，难免会需要根据条件来注册Bean。



例如，你想要根据不同的运行环境，来让Spring注册对应环境的数据源Bean，对于这种简单的情况，完全可以使用@Profile注解实现，就像下面代码所示：

```
1. @Configuration 

2. public class AppConfig { 

3.   @Bean 

4.   @Profile("DEV") 

5.   public DataSource devDataSource() { 

6.     ... 

7.   } 

8.    

9.   @Bean 

10.   @Profile("PROD") 

11.   public DataSource prodDataSource() { 

12.     ... 

13.   } 

14. } 
```



剩下只需要设置对应的Profile属性即可，设置方法有如下三种：

- 通过context.getEnvironment().setActiveProfiles("PROD")来设置Profile属性。
- 通过设定jvm的spring.profiles.active参数来设置环境（Spring Boot中可以直接在application.properties配置文件中设置该属性）。
- 通过在DispatcherServlet的初始参数中设置。

```
1. <servlet> 

2.   <servlet-name>dispatcher</servlet-name> 

3.   <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class> 

4.   <init-param> 

5.     <param-name>spring.profiles.active</param-name> 

6.     <param-value>PROD</param-value> 

7.   </init-param> 

8. </servlet> 
```



但这种方法只局限于简单的情况，而且通过源码我们可以发现@Profile自身也使用了@Conditional注解。

```
1. package org.springframework.context.annotation; 

2.  

3. @Target({ElementType.TYPE, ElementType.METHOD}) 

4. @Retention(RetentionPolicy.RUNTIME) 

5. @Documented 

6. @Conditional({ProfileCondition.class}) // 组合了Conditional注解 

7. public @interface Profile { 

8.   String[] value(); 

9. } 

10.  

11. package org.springframework.context.annotation; 

12.  

13. class ProfileCondition implements Condition { 

14.   ProfileCondition() { 

15.   } 

16.  

17.   // 通过提取出@Profile注解中的value值来与profiles配置信息进行匹配 

18.   public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) { 

19.     if(context.getEnvironment() != null) { 

20.       MultiValueMap attrs = metadata.getAllAnnotationAttributes(Profile.class.getName()); 

21.       if(attrs != null) { 

22.         Iterator var4 = ((List)attrs.get("value")).iterator(); 

23.  

24.         Object value; 

25.         do { 

26.           if(!var4.hasNext()) { 

27.             return false; 

28.           } 

29.  

30.           value = var4.next(); 

31.         } while(!context.getEnvironment().acceptsProfiles((String[])((String[])value))); 

32.  

33.         return true; 

34.       } 

35.     } 

36.  

37.     return true; 

38.   } 

39. } 
```



在业务复杂的情况下，显然需要使用到@Conditional注解来提供更加灵活的条件判断，例如以下几个判断条件（一一对应基于@Conditional的各种情况）

- 在类路径中是否存在这样的一个类。
- 在Spring容器中是否已经注册了某种类型的Bean（如未注册，我们可以让其自动注册到容器中，上一条同理）。
- 一个文件是否在特定的位置上。
- 一个特定的系统属性是否存在。
- 在Spring的配置文件中是否设置了某个特定的值。



举个栗子，假设我们有两个基于不同数据库实现的DAO，它们全都实现了UserDao，其中JdbcUserDAO与MySql进行连接，MongoUserDAO与MongoDB进行连接。现在，我们有了一个需求，需要根据命令行传入的系统参数来注册对应的UserDao，就像java -jar app.jar -DdbType=MySQL会注册JdbcUserDao，而java -jar app.jar -DdbType=MongoDB则会注册MongoUserDao。使用@Conditional可以很轻松地实现这个功能，仅仅需要在你自定义的条件类中去实现Condition接口，让我们来看下面的代码。

```
1. public interface UserDAO { 

2.   .... 

3. } 

4.  

5. public class JdbcUserDAO implements UserDAO { 

6.   .... 

7. } 

8.  

9. public class MongoUserDAO implements UserDAO { 

10.   .... 

11. } 

12.  

13. public class MySQLDatabaseTypeCondition implements Condition { 

14.   @Override 

15.   public boolean matches(ConditionContext conditionContext, AnnotatedTypeMetadata metadata) { 

16.     String enabledDBType = System.getProperty("dbType"); // 获得系统参数 dbType 

17.     // 如果该值等于MySql，则条件成立 

18.     return (enabledDBType != null && enabledDBType.equalsIgnoreCase("MySql")); 

19.   } 

20. } 

21.  

22. // 与上述逻辑一致 

23. public class MongoDBDatabaseTypeCondition implements Condition { 

24.   @Override 

25.   public boolean matches(ConditionContext conditionContext, AnnotatedTypeMetadata metadata) { 

26.     String enabledDBType = System.getProperty("dbType"); 

27.     return (enabledDBType != null && enabledDBType.equalsIgnoreCase("MongoDB")); 

28.   } 

29. } 

30.  

31. // 根据条件来注册不同的Bean 

32. @Configuration 

33. public class AppConfig { 

34.   @Bean 

35.   @Conditional(MySQLDatabaseTypeCondition.class) 

36.   public UserDAO jdbcUserDAO() { 

37.     return new JdbcUserDAO(); 

38.   } 

39.    

40.   @Bean 

41.   @Conditional(MongoDBDatabaseTypeCondition.class) 

42.   public UserDAO mongoUserDAO() { 

43.     return new MongoUserDAO(); 

44.   } 

45. } 
```



现在，我们又有了一个新需求，我们想要根据当前工程的类路径中是否存在MongoDB的驱动类来确认是否注册MongoUserDAO。为了实现这个需求，可以创建检查MongoDB驱动是否存在的两个条件类。

```
1. public class MongoDriverPresentsCondition implements Condition { 

2.   @Override 

3.   public boolean matches(ConditionContext conditionContext, AnnotatedTypeMetadata metadata) { 

4.     try { 

5.       Class.forName("com.mongodb.Server"); 

6.       return true; 

7.     } catch (ClassNotFoundException e) { 

8.       return false; 

9.     } 

10.   } 

11. } 

12.  

13. public class MongoDriverNotPresentsCondition implements Condition { 

14.   @Override 

15.   public boolean matches(ConditionContext conditionContext, AnnotatedTypeMetadata metadata) { 

16.     try { 

17.       Class.forName("com.mongodb.Server"); 

18.       return false; 

19.     } catch (ClassNotFoundException e) { 

20.       return true; 

21.     } 

22.   } 

23. } 
```



假如，你想要在UserDAO没有被注册的情况下去注册一个UserDAOBean，那么我们可以定义一个条件类来检查某个类是否在容器中已被注册。

```
1. public class UserDAOBeanNotPresentsCondition implements Condition { 

2.   @Override 

3.   public boolean matches(ConditionContext conditionContext, AnnotatedTypeMetadata metadata) { 

4.     UserDAO userDAO = conditionContext.getBeanFactory().getBean(UserDAO.class); 

5.     return (userDAO == null); 

6.   } 

7. } 
```



如果你想根据配置文件中的某项属性来决定是否注册MongoDAO，例如app.dbType是否等于MongoDB，我们可以实现以下的条件类。

```
1. public class MongoDbTypePropertyCondition implements Condition { 

2.   @Override 

3.   public boolean matches(ConditionContext conditionContext, AnnotatedTypeMetadata metadata) { 

4.     String dbType = conditionContext.getEnvironment().getProperty("app.dbType"); 

5.     return "MONGO".equalsIgnoreCase(dbType); 

6.   } 

7. } 
```



我们已经尝试并实现了各种类型的条件判断，接下来，我们可以选择一种更为优雅的方式，就像@Profile一样，以注解的方式来完成条件判断。首先，我们需要定义一个注解类。

```
1. @Target({ ElementType.TYPE, ElementType.METHOD }) 

2. @Retention(RetentionPolicy.RUNTIME) 

3. @Documented 

4. @Conditional(DatabaseTypeCondition.class) 

5. public @interface DatabaseType { 

6.   String value(); 

7. } 
```



具体的条件判断逻辑在DatabaseTypeCondition类中，它会根据系统参数dbType来判断注册哪一个Bean。

```
1. public class DatabaseTypeCondition implements Condition { 

2.   @Override 

3.   public boolean matches(ConditionContext conditionContext, AnnotatedTypeMetadata metadata) { 

4.     Map<String, Object> attributes = metadata 

5.                       .getAnnotationAttributes(DatabaseType.class.getName()); 

6.     String type = (String) attributes.get("value"); 

7.     // 默认值为MySql 

8.     String enabledDBType = System.getProperty("dbType", "MySql"); 

9.     return (enabledDBType != null && type != null && enabledDBType.equalsIgnoreCase(type)); 

10.   } 

11. } 
```



最后，在配置类应用该注解即可。

```
1. @Configuration 

2. @ComponentScan 

3. public class AppConfig { 

4.   @Bean 

5.   @DatabaseType("MySql") 

6.   public UserDAO jdbcUserDAO() { 

7.     return new JdbcUserDAO(); 

8.   } 

9.  

10.   @Bean 

11.   @DatabaseType("mongoDB") 

12.   public UserDAO mongoUserDAO() { 

13.     return new MongoUserDAO(); 

14.   } 

15. } 
```



基于@Conditional，SpringBoot已经提供了很多可以直接使用的注解

比如

- @ConditionalOnClass：只有类存在返回true
- @ConditionalOnProperty：只有配置文件中某个属性值存在返回true
- @ConditionalOnMissingBean：只有spring容器中不存在这个Bean才返回true
- @ConditionalOnWebApplication：只有当前应用是Web应用返回true



等等



我们打开@ConditionalOnClass的源码看下

```
1. @Target({ElementType.TYPE, ElementType.METHOD}) 

2. @Retention(RetentionPolicy.RUNTIME) 

3. @Documented 

4. @Conditional({OnClassCondition.class}) 

5. public @interface ConditionalOnClass { 

6.   Class<?>[] value() default {}; 

7.  

8.   String[] name() default {}; 

9. } 
```



重点判断OnClassCondition.class的返回结果是否为true



然后，我们开始看OnClassCondition的源码，发现它并没有直接实现Condition接口，只好往上找，发现它的父类SpringBootCondition实现了Condition接口。

```
1. class OnClassCondition extends SpringBootCondition implements AutoConfigurationImportFilter, BeanFactoryAware, BeanClassLoaderAware { 

2.   ..... 

3. } 

4.  

5. public abstract class SpringBootCondition implements Condition { 

6.   private final Log logger = LogFactory.getLog(this.getClass()); 

7.  

8.   public SpringBootCondition() { 

9.   } 

10.  

11.   public final boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) { 

12.     String classOrMethodName = getClassOrMethodName(metadata); 

13.  

14.     try { 

15.       ConditionOutcome ex = this.getMatchOutcome(context, metadata); 

16.       this.logOutcome(classOrMethodName, ex); 

17.       this.recordEvaluation(context, classOrMethodName, ex); 

18.       return ex.isMatch(); 

19.     } catch (NoClassDefFoundError var5) { 

20.       throw new IllegalStateException("Could not evaluate condition on " + classOrMethodName + " due to " + var5.getMessage() + " not found. Make sure your own configuration does not rely on that class. This can also happen if you are @ComponentScanning a springframework package (e.g. if you put a @ComponentScan in the default package by mistake)", var5); 

21.     } catch (RuntimeException var6) { 

22.       throw new IllegalStateException("Error processing condition on " + this.getName(metadata), var6); 

23.     } 

24.   } 

25.  

26.   public abstract ConditionOutcome getMatchOutcome(ConditionContext var1, AnnotatedTypeMetadata var2); 

27. } 
```



SpringBootCondition实现的matches方法依赖于一个抽象方法this.getMatchOutcome(context, metadata)，我们在它的子类OnClassCondition中可以找到这个方法的具体实现，具体实现逻辑过于复杂就不仔细查看了。



**所以@Conditional类是一切的基础，追朔本源都是判断是否match。**