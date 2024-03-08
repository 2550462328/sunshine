NoSQL数据库的四大家族

- 列存储：Hbase
- 键值(Key-Value)存储：Redis
- 图像存储：Neo4J
- 文档存储：MongoDB



#### 1. MongoDB 和RDBMS(关系型数据库)对比

MongoDB是一个介于关系数据库和非关系数据库之间的产品，是非关系数据库当中功能最丰富，最像关系数据库的。它支持的数据结构非常松散，是类似json的bson格式，因此可以存储比较复杂的数据类型。

![img](http://pcc.huitogo.club/7b241a376cc749522f909bcaf01e0691)



- database 数据库，与SQL的数据库(database)概念相同，一个数据库包含多个集合(表)
- collection 集合，相当于SQL中的表(table)，一个集合可以存放多个文档(行)。 不同之处就在于集合的结构(schema)是动态的，不需要预先声明一个严格的表结构。更重要的是，默认情况下MongoDB 并不会对写入的数据做任何schema的校验。
- document 文档，相当于SQL中的行(row)，一个文档由多个字段(列)组成，并采用bson(json)格式表示。
- field 字段，相当于SQL中的列(column)，相比普通column的差别在于field的类型可以更加灵活，比如支持嵌套的文档、数组。



此外，MongoDB中字段的类型是固定的、区分大小写、并且文档中的字段也是有序的。



**什么是BSON数据类型？**

BSON是一种类json的一种二进制形式的存储格式，简称Binary JSON，它和JSON一样，支持内嵌的文档对象和数组对象，但是BSON有JSON没有的一些数据类型，如Date和Binary Data类型。BSON可以做为网络数据交换的一种存储形式,是一种schema-less的存储形式，它的优点是灵活性高，但它的缺点是空间利用率不是很理想。

```
{
"patientName": "张三",
"sexCode": "1",
"age": 27,
"mainSuit": "头晕一周",
"illnessHistory": "头晕一周",
"deptName": "急诊科",
"diagList": [{
"diagnosticCode": "BEZ030",
"diagnosticName": "哮喘病",
"diagnosticMainSign": "1",
"diagTypeCode": "1"
}]
}
```



MongoDB中Document中可以使用的数据类型：

![img](http://pcc.huitogo.club/15469efee0cc3baa53589acae22f8ae7)



#### 2. Mongodb使用场景

为WEB应用提供可扩展、高性能、易部署的数据存储解决方案

它是一个内存数据库，对数据的操作大部分都在内存中，mongodb的所有数据实际上是存放在硬盘的，所有要操作的数据通过mmap的方式映射到内存某个区域内。然后，mongodb就在这块区域里面进行数据修改，避免了零碎的硬盘操作。



MongoDB的适用场景

- 网站数据：Mongo 非常适合实时的插入,更新与查询，并具备网站实时数据存储所需的复制及高度伸缩性。
- 缓存：由于性能很高，Mongo 也适合作为信息基础设施的缓存层。在系统重启之后，由Mongo搭建的持久化缓存层可以避免下层的数据源过载。
- 大尺寸、低价值的数据：使用传统的关系型数据库存储一些大尺寸低价值数据时会比较浪费在此之前，很多时候程序员往往会选择传统的文件进行存储。
- 高伸缩性的场景：Mongo 非常适合由数十或数百台服务器组成的数据库，Mongo 的路线图中已经包含对MapReduce 引擎的内置支持以及集群高可用的解决方案。
- 用于对象及JSON数据的存储：Mongo的BSON数据格式非常适合文档化格式的存储及查询。



MongoDB的行业具体应用场景

MongoDB 的应用已经渗透到各个领域，比如游戏、物流、电商、内容管理、社交、物联网、视频直播等，以下是几个实际的应用案例：

1. 游戏场景，使用 MongoDB 存储游戏用户信息，用户的装备、积分等直接以内嵌文档的形式存储，方便查询、更新。
2. 物流场景，使用 MongoDB 存储订单信息，订单状态在运送过程中会不断更新，以 MongoDB 内嵌数组的形式来存储，一次查询就能将订单所有的变更读取出来。
3. 社交场景，使用 MongoDB 存储存储用户信息，以及用户发表的朋友圈信息，通过地理位置索引实现附近的人、地点等功能
4. 物联网场景，使用 MongoDB 存储所有接入的智能设备信息，以及设备汇报的日志信息，并对这些信息进行多维度的分析
5. 视频直播，使用 MongoDB 存储用户信息、礼物信息等 ......



**如何抉择是否使用MongoDB**

![img](http://pcc.huitogo.club/70188c48aa7f27cfc55eece352eaed4b)