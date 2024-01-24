**Spring的事务是通过AOP代理类中的一个Advice（TransactionInterceptor）进行生效的**，AOP是通过预编译和动态代理方式实现了，java是使用的动态代理模式(JDK+CGLIB)。

在了解Spring代理的两种特点后，我们也就知道在做事务切面配置时的一些注意事项，例如JDK代理时**方法必须是public**，CGLIB代理时必须是public、protected，且**类不能是final的**；在依赖注入时，如果属性类型定义为实现类，JDK代理时会报UnsatisfiedDependencyException异常，但如果修改为CGLIB代理时则会成功注入，所以**如果有接口，建议注入时该类属性都定义为接口。另外事务切点都配置在实现类和接口都可以生效，但建议加在实现类上**。



说完spring的事务实现机制，开始讲讲坑吧

Spring事务在哪些情况下会失效呢？



#### 1. 没有被Spring管理

```
1. // @Service 

2. public class OrderServiceImpl implements OrderService {   

3. @Transactional   

4. public void updateOrder(Order order) {     

5. // update order；  

6.  } 

7. } 
```

如果此时把 @Service 注解注释掉，这个类就不会被加载成一个 Bean，那这个类就不会被 Spring 管理了，事务自然就失效了。



#### 2. 方法不是public

```
1. @Service 

2. public class DemoServiceImpl implements DemoService { 

3.  

4.   @Transactional(rollbackFor = SQLException.class) 

5.   @Override 

6.   private int saveAll(){ // 编译器一般都会在这个地方给出错误提示 

7.     // do someThing; 

8.     return 1; 

9.   } 

10. } 
```

为什么方法必须是public呢？因为spring实现事务的关键在于aop，aop实现的关键是动态代理，比如说CGLIB动态代理吧，它是根据现有类生成它的子类，然后达到增强的效果，然后你让我增强一个private的方法？那我的代理类能调用到它么？



#### 3. 自身调用

先看两个示例，思考下他们的事务能不能生效？

```
1. @Service 

2. public class OrderServiceImpl implements OrderService { 

3.   public void update(Order order) { 

4.     updateOrder(order); 

5.   } 

6.   @Transactional 

7.   public void updateOrder(Order order) { 

8.     // update order； 

9.   } 

10. } 
```

update方法上面没有加 @Transactional 注解，调用有 @Transactional 注解的 updateOrder 方法



```
1. @Service 

2. public class OrderServiceImpl implements OrderService { 

3.   @Transactional 

4.   public void update(Order order) { 

5.     updateOrder(order);  

6.  } 

7.   @Transactional(propagation = Propagation.REQUIRES_NEW) 

8.   public void updateOrder(Order order) { 

9.     // update order； 

10.   } 

11. } 
```

在 update 方法上加了 @Transactional，updateOrder 加了 REQUIRES_NEW 新开启一个事务



答案就是它们都不能生效

1. 很简单，还是因为代理，update方法是没有事务包围的，也就是不会去调用代理类，那么在它原有的类中调自身，自然不会有增强效果。
2. 如果updateOrder的事务时REQUIRES_NEW，当update执行它的时候，update自身事务是挂起的，只有当updateOrder的事务提交完毕，update才能执行，所以换句话说update报错了updateOrder是不会回滚的，因为它们是两个事务。



解决方法有两种

- 在当前类中注入自身，用注入的对象再调用updateOrder方法。
- 自觉使用代理类调用需要事务包围的方法，比如

```
// 主动使用代理类

1. ((IUserService)AopContext.currentProxy()).updateUser(user); 
```



#### 4. 事务管理器没有配对数据源

```
1. @Bean 

2. public PlatformTransactionManager transactionManager(DataSource dataSource) { 

3.   return new DataSourceTransactionManager(dataSource); 

4. } 
```

比如java事务是建立在数据库事务的基础上，数据源没有绑定或者绑定错了，肯定不会有事务。



#### 5. 方法不支持事务

```
1. @Service 

2. public class OrderServiceImpl implements OrderService { 

3.   @Transactional 

4.   public void update(Order order) { 

5.     updateOrder(order); 

6.   } 


7.   @Transactional(propagation = Propagation.NOT_SUPPORTED) 

8.   public void updateOrder(Order order) { 

9.     // update order； 

10.   } 

11. } 
```

Propagation.NOT_SUPPORTED： 表示不以事务运行，当前若存在事务则挂起



#### 6. 异常被吃了

```
1. @Service 

2. public class OrderServiceImpl implements OrderService { 

3.   @Transactional 

4.   public void updateOrder(Order order) { 

5.     try { 

6.       // update order; 

7.     }catch (Exception e){ 

8.       //do something; 

9.     } 

10.   } 

11. } 
```

异常被吃掉了，怎么会触发事务回滚呢？



解决方案有

- 在catch中手动抛出异常
- 在catch中，编程式手动实现事务回滚

```
// 手动回滚

1. TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();   
```

注意：尽管可以采用编程式方法回滚事务，但“回滚”只是事务的生命周期之一，所以要么编程实现事务的全部必要周期，要么仍要配置事务切点，即，将事务管理的其他周期交由Spring的标识！



#### 7. 异常类型错误或格式配置错误

```
1. @Service 

2. public class OrderServiceImpl implements OrderService { 

3. @Transactional 

4.  // @Transactional(rollbackFor = SQLException.class) 

5.   public void updateOrder(Order order) { 

6.     try {      // update order 

7.      }catch (Exception e){ 

8.      throw new Exception("更新错误");     

9.     }   

10.   } 

11. } 
```



只有出现异常，事务管理器才会触发并做出处理，在**默认下只对Error和RuntimeException才会触发异常，一般的Check异常不会发生回滚**。如果需要回滚，手动配置

```
//  配置需要回滚的异常

@Transactional(rollbackFor = {Exception.class, RuntimeException.class})  
```



那哪些是RuntimeException（Exception下RunTimeException的子类），哪些是CheckException（Exception的直接子类）?

check异常

![img](http://pcc.huitogo.club/ad6c67692d372b8bd9e533c25d12491c)



unCheck异常

![img](http://pcc.huitogo.club/46e1d6e4137fbfc376ff4ee75e13ea88)



注意：SQLException是属于Exception的直接子类，即Check异常，但是在Spring框架下，**所有SQL异常都被org.springframework重写为RuntimeException，事务因此也会发生回滚！**



#### 8. 多线程操作，或者绕开spring获取数据库的方式去获取数据库

```
1. 1. public abstract class DataSourceUtils {  

2.  

3. 2. …  

4.  

5. 3. public static Connection getConnection(DataSource dataSource) throws CannotGetJdbcConnectionException {  

6.  

7. 4.     try {  

8.  

9. 5.       return doGetConnection(dataSource);  

10.  

11. 6.     }  

12.  

13. 7.     catch (SQLException ex) {  

14.  

15. 8.       throw new CannotGetJdbcConnectionException("Could not get JDBC Connection", ex);  

16.  

17. 9.     }  

18.  

19. 10.   }  

20.  

21.  

22. 11. public static Connection doGetConnection(DataSource dataSource) throws SQLException {  

23.  

24. 12.     Assert.notNull(dataSource, "No DataSource specified");  

25.  

26. 13.     ConnectionHolder conHolder = (ConnectionHolder) TransactionSynchronizationManager.getResource(dataSource);  

27.  

28. 14.     if (conHolder != null && (conHolder.hasConnection() || conHolder.isSynchronizedWithTransaction())) {  

29.  

30. 15.       conHolder.requested();  

31.  

32. 16.       if (!conHolder.hasConnection()) {  

33.  

34. 17.         logger.debug("Fetching resumed JDBC Connection from DataSource");  

35.  

36. 18.         conHolder.setConnection(dataSource.getConnection());  

37.  

38. 19.       }  

39.  

40. 20.       return conHolder.getConnection();  

41.  

42. 21.     }  

43.  

44. 22. …  

45.  

46. 23. }  
```

TransactionSynchronizationManager是一个事务同步管理的核心类，它实现了事务同步管理的职能，包括记录当前连接持有connection holder。

Spring是通过TransactionSynchronizationManager类中线程变量来获取事务中数据库连接，所以如果是多线程调用或者绕过Spring获取数据库连接，都会导致Spring事务配置失效。



#### 9. Spring事务超时不准确或失效

超时发生在最后一次JdbcTemplate操作之后

通过非JdbcTemplate操作数据库，例如Mybatis