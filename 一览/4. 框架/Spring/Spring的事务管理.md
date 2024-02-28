#### 1. Spring的事务开放接口

Spring的事务管理高层抽象主要包括3个接口



##### 1.1 PlatformTransactionManager

平台事务管理器，Spring 事务策略的核心。

**Spring 并不直接管理事务，而是提供了多种事务管理器 。**



通过这个接口，Spring 为各个平台如 JDBC(DataSourceTransactionManager)、Hibernate(HibernateTransactionManager)、JPA(JpaTransactionManager)等都提供了对应的事务管理器，但是具体的实现就是各个平台自己的事情了。



PlatformTransactionManager 接口的具体实现如下:

![img](http://pcc.huitogo.club/4596dcca5a998f8f6ae242a4d9bb33ba)



其中一些主要实现的事务管理器说明

![img](http://pcc.huitogo.club/4925932ca189d182ee3e8537769b00d7)



PlatformTransactionManager接口中定义了三个方法：

```
1. public interface PlatformTransactionManager { 

2.   //获得事务 

3.   TransactionStatus getTransaction(@Nullable TransactionDefinition var1) throws TransactionException; 

4.   //提交事务 

5.   void commit(TransactionStatus var1) throws TransactionException; 

6.   //回滚事务 

7.   void rollback(TransactionStatus var1) throws TransactionException; 

8. } 
```

- PlatformTransactionManager 是负责事务管理的接口，一共有三个接口方法，分别负责事务的获得、提交、回滚。
- #getTransaction(TransactionDefinition definition)方法，根据事务定义 TransactionDefinition ，获得 TransactionStatus 。
  - 为什么不是创建事务呢？因为如果当前如果已经有事务，则不会进行创建，一般来说会跟当前线程进行绑定。如果不存在事务，则进行创建。
  - 为什么返回的是 TransactionStatus 对象？在 TransactionStatus 中，不仅仅包含事务属性，还包含事务的其它信息，例如是否只读、是否为新创建的事务等等。
  - 事务 TransactionDefinition 是什么？下面，也会详细解析 TransactionStatus 。
- #commit(TransactionStatus status)方法，根据 TransactionStatus 情况，提交事务。
  - 为什么根据 TransactionStatus 情况，进行提交？例如说，带@Transactional注解的的 A 方法，会调用 @Transactional注解的的 B 方法。
    - 在 B 方法结束调用后，会执行 `PlatformTransactionManager#commit(TransactionStatus status)` 方法，此处事务**是不能**、**也不会**提交的。
    - 而是在 A 方法结束调用后，执行 `PlatformTransactionManager#commit(TransactionStatus status)` 方法，提交事务。
- #rollback(TransactionStatus status)方法，根据 TransactionStatus 情况，回滚事务。
  - 为什么根据 TransactionStatus 情况，进行回滚？原因同 `#commit(TransactionStatus status)` 方法。



##### 1.2 TransactionDefinition

事务定义信息(事务隔离级别、传播行为、超时、只读、回滚规则)

事务管理器接口 PlatformTransactionManager 通过 getTransaction(TransactionDefinition definition) 方法来得到一个事务



事务属性包含了 5 个方面：

![img](http://pcc.huitogo.club/252cb247b17f51a06f08c2d4444ecfec)



TransactionDefinition 接口中定义了 5 个方法以及一些表示事务属性的常量比如隔离级别、传播行为等等。

```
1. public interface TransactionDefinition { 

2.   int PROPAGATION_REQUIRED = 0; 

3.   int PROPAGATION_SUPPORTS = 1; 

4.   int PROPAGATION_MANDATORY = 2; 

5.   int PROPAGATION_REQUIRES_NEW = 3; 

6.   int PROPAGATION_NOT_SUPPORTED = 4; 

7.   int PROPAGATION_NEVER = 5; 

8.   int PROPAGATION_NESTED = 6; 

9.   int ISOLATION_DEFAULT = -1; 

10.   int ISOLATION_READ_UNCOMMITTED = 1; 

11.   int ISOLATION_READ_COMMITTED = 2; 

12.   int ISOLATION_REPEATABLE_READ = 4; 

13.   int ISOLATION_SERIALIZABLE = 8; 

14.   int TIMEOUT_DEFAULT = -1; 

15.   // 返回事务的传播行为，默认值为 REQUIRED。 

16.   int getPropagationBehavior(); 

17.   //返回事务的隔离级别，默认值是 DEFAULT 

18.   int getIsolationLevel(); 

19.   // 返回事务的超时时间，默认值为-1。如果超过该时间限制但事务还没有完成，则自动回滚事务。 

20.   int getTimeout(); 

21.   // 返回是否为只读事务，默认值为 false 

22.   boolean isReadOnly(); 

23.  

24.   @Nullable 

25.   String getName(); 

26. } 
```



**事务只读属性**

对于只有读取数据查询的事务，可以指定事务类型为 readonly，即只读事务。只读事务不涉及数据的修改，数据库会提供一些优化手段，适合用在有多条数据库查询操作的方法中。



很多人就会疑问了，**为什么我一个数据查询操作还要启用事务支持呢**？



拿 MySQL 的 innodb 举例子，根据官网描述：

MySQL 默认对每一个新建立的连接都启用了autocommit模式。在该模式下，每一个发送到 MySQL 服务器的sql语句都会在一个单独的事务中进行处理，执行结束后会自动提交事务，并开启一个新的事务。



因此

- 如果你一次执行单条查询语句，则没有必要启用事务支持，数据库默认支持 SQL 执行期间的读一致性；
- 如果你一次执行多条查询语句，例如统计查询，报表查询，在这种场景下，多条查询 SQL 必须保证整体的读一致性，否则，在前条 SQL 查询之后，后条 SQL 查询之前，数据被其他用户改变，则该次整体的统计查询将会出现读数据不一致的状态，此时，应该启用事务支持



##### 1.3 TransactionStatus

**事务运行状态**

PlatformTransactionManager 会根据 TransactionDefinition 的定义比如事务超时时间、隔离级别、传播行为等来进行事务管理 ，而 TransactionStatus 接口则提供了一些方法来获取事务相应的状态比如是否新事务、是否可以回滚等等。

PlatformTransactionManager.getTransaction(…)方法返回一个 TransactionStatus 对象。



TransactionStatus 接口内容如下：

```
1. public interface TransactionStatus{ 

2.   boolean isNewTransaction(); // 是否是新的事物 

3.   boolean hasSavepoint(); // 是否有恢复点 

4.   void setRollbackOnly(); // 设置为只回滚 

5.   boolean isRollbackOnly(); // 是否为只回滚 

6.   boolean isCompleted; // 是否已完成 

7. } 
```



#### 2. Spring支持两种事务管理

##### 2.1 编程式事务管理

通过 TransactionTemplate或者TransactionManager手动管理事务，实际应用中很少使用



使用 TransactionTemplate进行编程式事务管理的示例代码如下：

```
1. @Autowired 

2. private TransactionTemplate transactionTemplate; 

3. public void testTransaction() { 

4.  

5.     transactionTemplate.execute(new TransactionCallbackWithoutResult() { 

6.       @Override 

7.       protected void doInTransactionWithoutResult(TransactionStatus transactionStatus) { 

8.  

9.         try { 

10.  

11.           // .... 业务代码 

12.         } catch (Exception e){ 

13.           //回滚 

14.           transactionStatus.setRollbackOnly(); 

15.         } 

16.  

17.       } 

18.     }); 

19. } 
```



使用 TransactionManager 进行编程式事务管理的示例代码如下：

```
1. @Autowired 

2. private PlatformTransactionManager transactionManager; 

3.  

4. public void testTransaction() { 

5.  

6.  TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition()); 

7.      try { 

8.        // .... 业务代码 

9.        transactionManager.commit(status); 

10.      } catch (Exception e) { 

11.        transactionManager.rollback(status); 

12.      } 

13. } 
```



##### 2.2 声明式事务管理

推荐使用（代码侵入性最小），原理是通过 AOP 实现



使用 @Transactional注解进行事务管理的示例代码如下：

```
1. @Transactional(propagation=propagation.PROPAGATION_REQUIRED) 

2. public void aMethod { 

3.  //do something 

4.  B b = new B(); 

5.  C c = new C(); 

6.  b.bMethod(); 

7.  c.cMethod(); 

8. } 
```



**@Transactional的原理**

如果一个类或者一个类中的 public 方法上被标注@Transactional 注解的话，Spring 容器就会在启动的时候为其创建一个代理类，在调用被@Transactional 注解的 public 方法的时候，实际调用的是，TransactionInterceptor 类中的 invoke()方法。这个方法的作用就是在目标方法之前开启事务，方法执行过程中如果遇到异常的时候回滚事务，方法调用完成之后提交事务。



TransactionInterceptor 类中的 invoke()方法内部实际调用的是 TransactionAspectSupport 类的 invokeWithinTransaction()方法。由于新版本的 Spring 对这部分重写很大，而且用到了很多响应式编程的知识，这里就不列源码了。