#### 1. 怎么解析的？

查找mybatis-config.xml文件，解析 标签中的子节点 到Configuration中

```
// 配置常量
propertiesElement(root.evalNode("properties")); 
// 配置别名
typeAliasesElement(root.evalNode("typeAliases"));
// 配置插件（拦截器）
pluginElement(root.evalNode("plugins"));
// 配置查询结果 转成对象的生成工厂（通过MetaObject）
objectFactoryElement(root.evalNode("objectFactory"));
objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
// 常规配置（如配置列表 通过驼峰+下划线转换到 字段名）
settingsElement(root.evalNode("settings"));
// 环境变量
environmentsElement(root.evalNode("environments"));
databaseIdProviderElement(root.evalNode("databaseIdProvider"));
// 出入参数类型转换
typeHandlerElement(root.evalNode("typeHandlers"));
// 核心 解析 *Mapper.xml 到MapperStatement
mapperElement(root.evalNode("mappers"));
```



Configuration中对应的属性

```
protected Environment environment;
protected Properties variables = new Properties();
protected ObjectFactory objectFactory = new DefaultObjectFactory();
protected ObjectWrapperFactory objectWrapperFactory = new DefaultObjectWrapperFactory();
protected MapperRegistry mapperRegistry = new MapperRegistry(this);
protected final InterceptorChain interceptorChain = new InterceptorChain();
protected final TypeHandlerRegistry typeHandlerRegistry = new TypeHandlerRegistry();
protected final TypeAliasRegistry typeAliasRegistry = new TypeAliasRegistry();
protected final LanguageDriverRegistry languageRegistry = new LanguageDriverRegistry();
protected final Map<String, MappedStatement> mappedStatements = new StrictMap<MappedStatement>("Mapped Statements collection");
protected final Map<String, Cache> caches = new StrictMap<Cache>("Caches collection");
protected final Map<String, ResultMap> resultMaps = new StrictMap<ResultMap>("Result Maps collection");
protected final Map<String, ParameterMap> parameterMaps = new StrictMap<ParameterMap>("Parameter Maps collection");
protected final Map<String, KeyGenerator> keyGenerators = new StrictMap<KeyGenerator>("Key Generators collection");
```



重点看一下怎么将*Mapper.xml 解析到 MapperStatements

```
String namespace = context.getStringAttribute("namespace");
if (namespace.equals("")) {
 throw new BuilderException("Mapper's namespace cannot be empty");
}
builderAssistant.setCurrentNamespace(namespace);
// MapperStatement缓存
cacheRefElement(context.evalNode("cache-ref"));
cacheElement(context.evalNode("cache"));
// 入参映射
parameterMapElement(context.evalNodes("/mapper/parameterMap"));
// 出参映射
resultMapElements(context.evalNodes("/mapper/resultMap"));
// sql 节点
sqlElement(context.evalNodes("/mapper/sql"));
// 封装MapperStatement
buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
```



- <parameterMap>标签会被解析为 ParameterMap 对象，其每个子元素会被解析为 ParameterMapping 对象。

- <resultMap>标签会被解析为 ResultMap 对象，其每个子元素会被解析为 ResultMapping 对象。
- <insert>、<select>、<update>、<delete>标签均会被解析为 MappedStatement 对象，标签内的 sql 会被解析为 BoundSql 对象。



在buildStatementFromContext过程中，也就是组装成下面的MapperStatement对象

```
// 原始 *Mapper.xml资源
private String resource;
private Configuration configuration;
private String id;
// fetchsize决定了每批次可以传输的记录条数，但同时，也决定了内存的大小
private Integer fetchSize;
private Integer timeout;
// 语句执行类型
private StatementType statementType;
// 返回结果集类型
private ResultSetType resultSetType;
// 待执行数据
private SqlSource sqlSource;
// 语句缓存
private Cache cache;
private ParameterMap parameterMap;
private List<ResultMap> resultMaps;
private boolean flushCacheRequired;
private boolean useCache;
private boolean resultOrdered;
private SqlCommandType sqlCommandType;
private KeyGenerator keyGenerator;
private String[] keyProperties;
private String[] keyColumns;
private boolean hasNestedResultMaps;
private String databaseId;
private Log statementLog;
private LanguageDriver lang;
private String[] resultSets;
```



在生成MapperStatement中，有几个重要点

mybatis 最终执行的sql字符串就是由SqlSource提供的，而mybatis是支持动态标签的，比如<where><foreach><trim>等，这部分标签会被解析成SqlNode

![img](http://pcc.huitogo.club/f0a2fe8d140bf15cb8ff99c39f4e8ac5)



这里LangugeDriver就是基于 静态sql 或者 动态 生成 SqlSource，目前支持XmlLanguageDriver 和 RawLanguageDriver

- XMLLanguageDriver:用于创建动态、静态SqlSource。

- RawLanguageDriver：在确保只有静态sql时，可以使用，不得含有任何动态sql的内容，否则，请使用XMLLanguageDriver。它其实是对XMLLanguageDriver创建的结果进行唯静态sql检查而已，发现有动态sql的内容，就抛异常。




解析的底层是使用Dom4j 把XML解析成一个个子节点，在通过 XMLScriptBuilder 遍历这些子节点最后生成对应的SqlSource。

![img](http://pcc.huitogo.club/1f787e20a09ae5e54c620d50931fbf32)





#### 2. 怎么执行的？

这里先说一点 ，mybatis 不具备执行sql 语句的能力，它只是对接 Jbdc规范（Connection、Statement、Transaction、ResultSet），然后 由各数据库产商去实现Jdbc 规范，所以mybatis 的目标是 包装成 Jdbc的规范

```
Statement stmt = null;
try {
  Configuration configuration = ms.getConfiguration();
  // 添加 Statement的 Plugin Handler
  StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
  stmt = prepareStatement(handler, ms.getStatementLog());
  // 执行 statement.execute()
  return handler.<E>query(stmt, resultHandler);
} finally {
  closeStatement(stmt);
}
```



这里再回顾一下mybatis 的三个组件的执行规则

```
SqlSessionFactoryBuilder  --合成-  Configuration ---解析- mybatis.config 

	生成（MappedStatement）XMLMapperBuilder.configurationElement() ---> XMLStatementBuilder.parseStatementNode

	KeyGenerator、StatementType、LanguageDriver（处理动态标签）、ParameterMap、ResultMap、ResultType
	

SqlSessionFactory

	DataSource、Transaction
	

SqlSession（执行sql 语句  ）

	Executor（SimpleExecutor、ReuseExecutor、BatchExecutor、ClosedExecutor）

	Statement.execute -- SimpleStatement、PrepareStatement、CallableStatement(支持调用存储过程,提供了对输出和输入/输出参数(INOUT)的支持; )

	TypeHandler ResultHandler  InterceptorChain（Plugin）
    
	RowBounds（分页）

    语句缓存  CacheKey
```



##### 2.1 Executor（语句执行器）

- SimpleExecutor：每执行一次 update 或 select，就开启一个 Statement 对象，用完立刻关闭 Statement 对象。

- ReuseExecutor：执行 update 或 select，以 sql 作为 key 查找 Statement 对象，存在就使用，不存在就创建，用完后，不关闭 Statement 对象，而是放置于 Map<String, Statement>内，供下一次使用。简言之，就是重复使用 Statement 对象。


```
BoundSql boundSql = handler.getBoundSql();
String sql = boundSql.getSql();
// 如果有 就不会重复创建Statement
if (hasStatementFor(sql)) {
  stmt = getStatement(sql);
} else {
  Connection connection = getConnection(statementLog);
  stmt = handler.prepare(connection);
  putStatement(sql, stmt);
}
```

- BatchExecutor：执行 update（没有 select，JDBC 批处理不支持 select），将所有 sql 都添加到批处理中（addBatch()），等待统一执行（executeBatch()），它缓存了多个 Statement 对象，每个 Statement 对象都是 addBatch()完毕后，等待逐一执行 executeBatch()批处理。与 JDBC 批处理相同。


```
Connection connection = getConnection(ms.getStatementLog());
stmt = handler.prepare(connection);
currentSql = sql;
currentStatement = ms;
statementList.add(stmt);
batchResultList.add(new BatchResult(ms, sql, parameterObject));
```



##### 2.2 PerpetualCache（语句缓存）

在执行sql 语句的时候，会有一个缓存机制，保证下次遇到相同的 Statement 不需要 在从数据库查询，这个是sqlSession级别的缓存，数据库层面也是有缓存的，基于PrepareStatement的缓存，如果预参数 相同，则不需要对sql进行二次校验 + 优化，直接执行



大概逻辑代码如下：

```
// 不需要对结果处理 可以试试先从缓存处理
list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
if (list != null) {
  handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
} else {
  list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
}
```



这里的localCache就是PerpetualCache，缓存key的重要组成部分 包括 MapperStatement的Id、分页参数、实际执行sql语句 和 经过TypeHandler转换之后的入参

```
CacheKey cacheKey = new CacheKey();
cacheKey.update(ms.getId());
cacheKey.update(rowBounds.getOffset());
cacheKey.update(rowBounds.getLimit());
cacheKey.update(boundSql.getSql());
...
 else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
     cacheKey.update(parameterObject);
 }
```



这里语句执行的话 有一个延迟加载机制，是使用 DeferredLoad 实现的，先看一下 queryFromDatabase语句

```
private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
  List<E> list;
  // 执行之前先放入占位
  localCache.putObject(key, EXECUTION_PLACEHOLDER);
  try {
    list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
  } finally {
    localCache.removeObject(key);
  }
  // 执行之后再放入真实数据
  localCache.putObject(key, list);
  if (ms.getStatementType() == StatementType.CALLABLE) {
    localOutputParameterCache.putObject(key, parameter);
  }
  return list;
}
```



可以看出在实际执行sql语句之前，是有个伪返回值的，这个设计防止多个线程同时执行相同的Statement语句的时候，只会有一个线程和数据库交互进行真实获取，其他线程得到的是伪返回值，那这么伪返回值怎么赋真实值呢，就需要看DeferredLoad.load()

```
public void load() {
  @SuppressWarnings( "unchecked" ) // we suppose we get back a List
  List<Object> list = (List<Object>) localCache.getObject(key);
  // 替换返回结果集
  Object value = resultExtractor.extractObjectFromList(list, targetType);
  resultObject.setValue(property, value);
}
```



##### 2.3 Plugin（插件）

Mybatis仅可以编写针对ParameterHandler、ResultSetHandler、StatementHandler、Executor这4种接口的插件，Mybatis使用JDK的动态代理，为需要拦截的接口生成代理对象以实现接口方法拦截功能，每当执行这4种接口对象的方法时，就会进入拦截方法，具体就是InvocationHandler的invoke()方法，当然，只会拦截那些你指定需要拦截的方法。



使用很简单，继承Inteceptor即可

```
public static class AlwaysMapPlugin implements Interceptor {
  public Object intercept(Invocation invocation) throws Throwable {
    return "Always";
  }

  public Object plugin(Object target) {
    return Plugin.wrap(target, this);
  }

  public void setProperties(Properties properties) {
  }
}
```



在解析mybatis-config.xml 会 将这些Interceptor 装入到 Configuration中，在doQuery执行语句的时候会依次执行 ParameterHandler、StatementHandler、ResultSetHandler

```
protected BaseStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
  this.configuration = mappedStatement.getConfiguration();
  this.executor = executor;
  this.mappedStatement = mappedStatement;
  this.rowBounds = rowBounds;
  ...
  this.parameterHandler = configuration.newParameterHandler(mappedStatement, parameterObject, boundSql);
  this.resultSetHandler = configuration.newResultSetHandler(executor, mappedStatement, rowBounds, parameterHandler, resultHandler, boundSql);
}

public ParameterHandler newParameterHandler(MappedStatement mappedStatement, Object parameterObject, BoundSql boundSql) {
  ParameterHandler parameterHandler = mappedStatement.getLang().createParameterHandler(mappedStatement, parameterObject, boundSql);
  parameterHandler = (ParameterHandler) interceptorChain.pluginAll(parameterHandler);
  return parameterHandler;
}

public ResultSetHandler newResultSetHandler(Executor executor, MappedStatement mappedStatement, RowBounds rowBounds, ParameterHandler parameterHandler,
    ResultHandler resultHandler, BoundSql boundSql) {
  ResultSetHandler resultSetHandler = new DefaultResultSetHandler(executor, mappedStatement, parameterHandler, resultHandler, boundSql, rowBounds);
  resultSetHandler = (ResultSetHandler) interceptorChain.pluginAll(resultSetHandler);
  return resultSetHandler;
}
```





#### 3. Spring中怎么运行的

**MapperProxy** 和 **MapperProxyFactory**



在Spring中 需要借助 Spring 的 扫描机制 将 Mybatis的特定注解 注册到BeanFactory中，比如@Mapper

![img](http://pcc.huitogo.club/8509fea79dfaae019bdfc9bc2a890526)

```
MapperScanConfigurer.postProcessBeanDefinitionRegistry   spring的BeanPostProcessor

	ClassPathBeanDefinitionScanner.scan
	
		ClassPathMapperScanner.doScan
```



最终 将指定dao包 路径下的Mapper 转换成BeanDefinition 存储在BeanFactory中

```
private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
  for (BeanDefinitionHolder holder : beanDefinitions) {
    definition = (GenericBeanDefinition) holder.getBeanDefinition();
    String beanClassName = definition.getBeanClassName();
    definition.getConstructorArgumentValues().addGenericArgumentValue(beanClassName); 
    definition.setBeanClass(this.mapperFactoryBeanClass);
    definition.getPropertyValues().add("addToConfig", this.addToConfig);
    definition.getPropertyValues().add("sqlSessionFactory",
    definition.getPropertyValues().add("sqlSessionTemplate",
    definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
    definition.setLazyInit(lazyInitialization);
  }
}
```



每个BeanDefinition存储了 SqlSessionFactory的里面（可以找到数据源信息 和 MapperStatement），这里需要注意的是BeanClass 其实已经变成了 MapperFactoryBean

相当于你实际使用的时候，拿到的是一个MapperProxy，由MapperProxyFactory生成的，而MapperProxy 会帮助你去使用SqlSessionFactoty 生成SqlSession，从MapperStament中获取SqlSource 进行执行

```
public <T> T getMapper(Class<T> type) {
  return getConfiguration().getMapper(type, this);
}

public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    final MybatisMapperProxyFactory<T> mapperProxyFactory = (MybatisMapperProxyFactory<T>) knownMappers.get(type);
    return mapperProxyFactory.newInstance(sqlSession);
}

public T newInstance(SqlSession sqlSession) {
    final MybatisMapperProxy<T> mapperProxy = new MybatisMapperProxy<>(sqlSession, mapperInterface, methodCache);
    return return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[]{mapperInterface}, mapperProxy);
}
```





#### 4. SpringBoot中怎么运行的

运用了SpringBoot的自动装配

![img](http://pcc.huitogo.club/2d90ea145fa45e124f67bcdddca6e0a4)



引入了 MybatisAutoConfiguration（这里以MybatisPlus为例），原理是一样的

```
@Configuration
@ConditionalOnClass({SqlSessionFactory.class, SqlSessionFactoryBean.class})
@ConditionalOnSingleCandidate(DataSource.class)
@EnableConfigurationProperties({MybatisPlusProperties.class})
@AutoConfigureAfter({DataSourceAutoConfiguration.class, MybatisPlusLanguageDriverAutoConfiguration.class})
public class MybatisPlusAutoConfiguration implements InitializingBean {
}

@Configuration
@Import({MybatisPlusAutoConfiguration.AutoConfiguredMapperScannerRegistrar.class})
@ConditionalOnMissingBean({MapperFactoryBean.class, MapperScannerConfigurer.class})
public static class MapperScannerRegistrarNotFoundConfiguration implements InitializingBean {
}
```