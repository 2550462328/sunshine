Mybatis缓存机制示意图

![img](http://pcc.huitogo.club/e74159071534c8226f75077016f5aeeb)



#### 1. 一级缓存

在应用运行过程中，我们有可能在一次数据库会话中，执行多次查询条件完全相同的 SQL， MyBatis 提供了一级缓存的方案优化这部分场景，如果是相同的 SQL 语句，会优先命中一级缓存， 避免直接对数据库进行查询，提高性能。具体执行过程如下：

![img](http://pcc.huitogo.club/9bf9363503a4075d9aa7bfe076541e10)



每个 SqlSession 中持有了 Executor，每个 Executor 中有一个 LocalCache。当用户发起查询时， MyBatis 根据当前执行的语句生成 MappedStatement，在 Local Cache 进行查询，如果缓存命中 的话，直接返回结果给用户，如果缓存没有命中的话，查询数据库，结果写入 Local Cache，最后返回结果给用户。



具体实现类的类关系图如下图所示：

![img](https://awps-assets.meituan.net/mit-x/blog-images-bundle-2018a/d76ec5fe.jpg)

一级缓存执行的时序图：

![img](https://awps-assets.meituan.net/mit-x/blog-images-bundle-2018a/bb851700.png)

总结：

1. MyBatis 一级缓存的生命周期和 SqlSession 一致。
2. MyBatis 一级缓存内部设计简单，只是一个没有容量限定的 HashMap，在缓存的功能性上有所欠缺。
3. MyBatis 的一级缓存最大范围是 SqlSession 内部，有多个 SqlSession 或者分布式的环境下，数据库写操作会引起脏数据，建议设定缓存级别为 Statement。



下面我们来验证一下：

先准备一下mybatis-config文件（配置数据库、映射关系、实体别名等）

```
1.  <configuration>  

2.      <properties resource="dbconfig.properties" />  

3.      <settings>  

5.          <setting name="useGeneratedKeys" value="true" />  

6.          <setting name="defaultExecutorType" value="REUSE" />  

7.          <setting name="logImpl" value="STDOUT_LOGGING" />  

8.      </settings>  

9.      <typeAliases>  

10.         <typeAlias type="cn.ictt.entity.system.User" alias="User" />  

11.         <typeAlias type="cn.ictt.util.page.Page" alias="Page" />  

12.     </typeAliases>  

17.     <plugins>  

18.         <plugin interceptor="cn.ictt.plugin.PagePlugin">  

19.             <property name="dialect" value="mysql" />  

20.             <property name="pageSqlId" value=".*listPage.*" />  

21.         </plugin>  

22.     </plugins>  

23.     <environments default="mysql_developer">  

24.         <environment id="mysql_developer">  

25.             <transactionManager type="jdbc" />  

26.             <dataSource type="pooled">  

27.                 <property name="driver" value="${driverClassName}" />  

28.                 <property name="url" value="${url}" />  

29.                 <property name="username" value="${username}" />  

30.                 <property name="password" value="${password}" />  

31.             </dataSource>  

32.         </environment>  

33.     </environments>  

34.     <mappers>  

35.         <mapper resource="system/UserMapper.xml" />  

36.     </mappers>  

37. </configuration>
```



配置要注意先后关系，UserMapper.xml和UserMapper.java这里不写上了。

测试：

```
1.          InputStream resourceAsStream = Resources.getResourceAsStream("mybatis-config2.xml");  

2.          SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(resourceAsStream);  

3.          SqlSession sqlSession = sqlSessionFactory.openSession();  

4.          UserMapper userMapper =sqlSession.getMapper(UserMapper.class);  

5.          User user1 = userMapper.getById(1);  

6.          System.out.println(user1);  

7.  //      userMapper.editById(new User(1, "张辉", "123", 2));  

8.  //      userMapper.save(new User(2, "张辉", "123", 2));  

9.  //      sqlSession.commit();  

10. //      userMapper.deleteById(12);  

11.         User user2 = userMapper.getById(2);  

12.         System.out.println(user2);  

13.         sqlSession.close();  
```



总结一级缓存过程：

第一次发出一个查询 sql，sql 查询结果写入 sqlsession 的一级缓存中，缓存使用的数据结构是一 个map。

- key：MapperID+offset+limit+Sql+所有的入参
- value：用户信息

同一个 sqlsession 再次发出相同的 sql，就从缓存中取出数据。如果两次中间出现 **commit 操作** （修改、添加、删除），本 sqlsession 中的一级缓存区域全部清空，下次再去缓存中查询不到所以要从数据库查询，从数据库查询到再写入缓存，这样做的目的为了让缓存中存储的是最新的信息，**避免脏读**。



#### 2. 二级缓存

二级缓存是mapper级别的缓存，多个SqlSession去操作同一个Mapper的sql语句，多个SqlSession可以共用二级缓存，也就是说多个sqlSession可以共享一个mapper中的二级缓存区域，并且如果两个mapper的namespace相同，即使是两个mapper，那么这两个mapper中执行sql查询到的数据也将存在相同的二级缓存区域中。



具体的工作流程如下所示

![img](http://pcc.huitogo.club/286fabffb6fae4a5a31d5474f44927f1)

- mybatis的二级缓存是通过CacheExecutor实现的。

- CacheExecutor其实是 Executor 的代理对象。所有的查询操作，在 CacheExecutor 中都会先匹配缓存中是否存 在，不存在则查询数据库。

- key：MapperID+offset+limit+Sql+所有的入参




##### 2.1 单服务器缓存

在mybatis.config.xml中加入

```
1.  <!--开启二级缓存  -->  

2.  <settings>  

3.      <setting name="cacheEnabled" value="true"/>  

4.  </settings> 
```



在需要开启缓存的mapper.xml中加入

```
1.  <!-- 开启二级缓存 -->  

2.  <cache></cache> 
```



需要注意的是**需要将实体类（比如这里的User）进行序列化**

为了将缓存数据取出执行反序列化操作，因为二级缓存数据存储介质多种多样，不一定只存在内存中，有可能存在硬盘中，如果我们要再取这个缓存的话，就需要反序列化了。所以mybatis中的pojo都去实现Serializable接口。



下面是我的测试代码：

```
1.          InputStream resourceAsStream = Resources.getResourceAsStream("mybatis-config2.xml");  

2.          SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(resourceAsStream);  

3.          SqlSession sqlSession = sqlSessionFactory.openSession();  

4.          UserMapper userMapper =sqlSession.getMapper(UserMapper.class);  

5.          SqlSession sqlSession2 = sqlSessionFactory.openSession();  

6.          UserMapper userMapper2 =sqlSession2.getMapper(UserMapper.class);  

7.          SqlSession sqlSession3 = sqlSessionFactory.openSession();  

8.          UserMapper userMapper3 =sqlSession2.getMapper(UserMapper.class);  

9.          User user1 = userMapper.getById(2);  

10.         System.out.println(user1);  

11. //      userMapper.editById(new User(2, "张辉", "123", 2));  

12.         sqlSession.commit();  

13. //      sqlSession.close();  

14. //      userMapper.editById(new User(1, "张辉", "123", 2));  

15. //      userMapper3.save(new User(3, "张辉", "123", 2));  

16. //        

17. //      userMapper3.deleteById(1);  

18.         userMapper3.editById(new User(1, "张辉", "123", 2));  

19. //      sqlSession3.commit();  

20.         User user2 = userMapper2.getById(2);  

21.         System.out.println(user2);  

22.         sqlSession2.close();
```



测试结果：

1. 对于sqlSession1、sqlSession2，只有在sqlSession1.commit()或者sqlSession1.close()之后sqlSession2才能取到sqlSession1执行的缓存实体类。
2. 就算sqlSession1.close()了，如果有sqlSession3或者其它的sqlSession执行了update或者delete方法（不需要commit()），当前实体的缓存就被清除了。



总结：这种单服务器下的基于namespace的缓存，确实是对所以sqlSession都是可见的，**前提是提供缓存的sqlSession放弃对缓存的控制且在下一次做相同操作前都没有更新操作**。



如果需要设置mapper.xml中某个方法不需要使用缓存时，可以设置useCache="false"

```
1.  <select id="selectUserByUserId" useCache="false" resultType="com.ys.twocache.User" parameterType="int">  

2.      select * from user where id=#{id}  

3.  </select>  
```



如果希望某个方法执行完自动刷新缓存，可以设置flushCache=”true”

```
1.  <select id="selectUserByUserId" flushCache="true" resultType="com.ys.twocache.User" parameterType="int">  

2.      select * from user where id=#{id}  

3.  </select>  
```

默认commit后会执行一次flush操作。



##### 2.2 多服务器(集群)下的缓存

![img](http://pcc.huitogo.club/110899e8b9647d83684b120b35be6ce8)



MyBatis默认有对ehcache的支持，启用流程如下：

1）配置mybatis-ehcache的依赖。

2）在mybatis-config.xml中配置全局缓存

```
<!--开启二级缓存  -->  
<settings>  

      <setting name="cacheEnabled" value="true"/>  

  </settings>  
```



3）在UserMapper.xml中配置缓存类型

```
  <!-- 开启本mapper的namespace下的二级缓存   

      type:指定cache接口的实现类的类型，不写type属性，mybatis默认使用PerpetualCache  

     要和ehcache整合，需要配置type为ehcache实现cache接口的类型  

 -->  
 <cache type="org.mybatis.caches.ehcache.EhcacheCache"></cache>  
```



4）在classpath下配置ehcache.xml的配置来配置缓存

```
14. <?xml version="1.0" encoding="UTF-8"?>  

15. <ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  

16.     xsi:noNamespaceSchemaLocation="../config/ehcache.xsd">  

17.     <diskStore path="F:developehcache"/>  

18.     <defaultCache  

19.             maxElementsInMemory="10000"  

20.             eternal="false"  

21.             timeToIdleSeconds="120"  

22.             timeToLiveSeconds="120"  

23.             maxElementsOnDisk="10000000"  

24.             diskExpiryThreadIntervalSeconds="120"  

25.             memoryStoreEvictionPolicy="LRU">  

26.         <persistence strategy="localTempSwap"/>  

27.     </defaultCache>  

28. </ehcache> 
```

上述配置中：

- **diskStore**：指定数据在磁盘中的存储位置。
- **defaultCache**：当借助CacheManager.add("demoCache")创建Cache时，EhCache便会采用指定的的管理策略



以下属性是必须的：

- **maxElementsInMemory** ：在内存中缓存的element的最大数目
- **maxElementsOnDisk** ：在磁盘上缓存的element的最大数目，若是0表示无穷大
- eternal ：设定缓存的elements是否永远不过期。如果为true，则缓存的数据始终有效，如果为false那么还要根据timeToIdleSeconds，timeToLiveSeconds判断
- **overflowToDisk** ： 设定当内存缓存溢出的时候是否将过期的element缓存到磁盘上



以下属性是可选的：

- **timeToIdleSeconds** ：当缓存在EhCache中的数据前后两次访问的时间超过timeToIdleSeconds的属性取值时，这些数据便会删除，默认值是0,也就是可闲置时间无穷大
- **timeToLiveSeconds** ：缓存element的有效生命期，默认是0.,也就是element存活时间无穷大
- **diskSpoolBufferSizeMB**：这个参数设置DiskStore(磁盘缓存)的缓存区大小.默认是30MB.每个Cache都应该有自己的一个缓冲区
- **diskPersistent** ：在VM重启的时候是否启用磁盘保存EhCache中的数据，默认是false
- **diskExpiryThreadIntervalSeconds** ：磁盘缓存的清理线程运行间隔，默认是120秒。每个120s，相应的线程会进行一次EhCache中数据的清理工作
- **memoryStoreEvictionPolicy**：当内存缓存达到最大，有新的element加入的时候，移除缓存中element的策略。默认是LRU（最近最少使用），可选的有LFU（最不常使用）和FIFO（先进先出）



#### 3. 缓存比较

当开启缓存后，数据的查询执行的流程为：

**二级缓存 -> 一级缓存 -> 数据库**

1. MyBatis 的二级缓存相对于一级缓存来说，实现了 SqlSession 之间缓存数据的共享，同时粒度 更加细，能够到 namespace 级别，通过 Cache 接口实现类不同的组合，对 Cache 的可控性也更强。
2. MyBatis 在多表查询时，极大可能会出现脏数据，有设计上的缺陷，安全使用二级缓存的条件 比较苛刻。
3. 在分布式环境下，由于默认的 MyBatis Cache 实现都是基于本地的，分布式环境下必然会出现 读取到脏数据，需要使用集中式缓存将 MyBatis 的 Cache 接口实现，有一定的开发成本，直 接使用 Redis、Memcached 等分布式缓存可能成本更低，安全性也更高。



#### 4. 总结

对于访问多的查询请求且用户对查询结果实时性要求不高，此时可采用mybatis二级缓存技术降低数据库访问量，提高访问速度，业务场景比如：耗时较高的统计分析sql、电话账单查询sql等。

实现方法如下：通过设置刷新间隔时间，由mybatis每隔一段时间自动清空缓存，根据数据变化频率设置缓存刷新间隔flushInterval，比如设置为30分钟、60分钟、24小时等，根据需求而定。

mybatis二级缓存对细粒度的数据级别的缓存实现不好，比如如下需求：对商品信息进行缓存，由于商品信息查询访问量大，但是要求用户每次都能查询最新的商品信息，此时如果使用mybatis的二级缓存就无法实现当一个商品变化时只刷新该商品的缓存信息而不刷新其它商品的信息，因为mybaits的二级缓存区域以mapper为单位划分的，当一个商品信息变化会将所有商品信息的缓存数据全部清空。解决此类问题可能需要在业务层根据需求对数据有针对性缓存。