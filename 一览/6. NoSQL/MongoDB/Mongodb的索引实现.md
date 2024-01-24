#### 1. 索引类型

##### 1.1 单键索引 (Single Field)

MongoDB支持所有数据类型中的单个字段索引，并且可以在文档的任何字段上定义。

对于单个字段索引，索引键的排序顺序无关紧要，因为MongoDB可以在任一方向读取索引。

```
db.集合名.createIndex({"字段名":排序方式})

或

db.集合名.ensureIndex({"字段名":排序方式})
```



##### 1.2 复合索引(Compound Index）

通常我们需要在多个字段的基础上搜索表/集合，这是非常频繁的。 如果是这种情况，我们可能会考虑在MongoDB中制作复合索引。 复合索引支持基于多个字段的索引，这扩展了索引的概念并将它们扩展到索引中的更大域。



制作复合索引时要注意的重要事项包括：字段顺序与索引方向。

```
db.集合名.createIndex( { "字段名1" : 排序方式, "字段名2" : 排序方式 } )
db.medical_info.createIndex({"name":1,"age":1}
```



##### 1.3 多键索引（Multikey indexes）

针对属性包含数组数据的情况，MongoDB支持针对数组中每一个element创建索引，Multikey indexes支持strings，numbers和nested documents

```
db.集合名.createIndex({"字段名":排序方式})
db.medical_info.createIndex({"drugDetailLists":1})
```



##### 1.4 地理空间索引（Geospatial Index）

针对地理空间坐标数据创建索引。

```
db.city.insert({s
location : { type: "Point", coordinates: [117.27, 31.86]},
name: "合肥"
})
```



**GeoJSON**

MongoDB支持以下类型的GeoJSON对象类型：

- 点（Point）
- 线（LineString）
- 多边形（Polygon）
- 多点（MultiPoint）
- 多线（MultiLineString）
- 多个多边形（MultiPolygon）
- 几何集合（GeometryCollection）



要存储GeoJSON数据的话，在文档中使用 type字段来指定GeoJSON对象类型以及 coordinates 对象来指定对象的坐标

```
{ type: "<GeoJSON type>" , coordinates: <coordinates> }

//示例
{ "type": "Point", "coordinates": [lon(经度),lat(纬度)]}
```



地理位置索引的分类

- 2dsphere索引，用于存储和查找球面上的点
- 2d索引，用于存储和查找平面上的点



示例：

```
//创建2dsphere索引
db.city.createIndex({"location":"2dsphere"})
//查询离合肥1个度的城市
db.city.find({
"location":{
$geoWithin: {
$center: [ [ 117.27, 31.86 ] , 1]
}
}})
```



##### 1.5 全文索引

MongoDB提供了针对string内容的文本查询，Text Index支持任意属性值为string或string数组元素的索引查询。

一个集合仅支持最多一个Text Index，中文分词不理想

```
db.集合.createIndex({"字段": "text"})
```



##### 1.6 哈希索引 Hashed Index

针对属性的哈希值进行索引查询，hash index仅支持等于查询，不支持范围查询。

```
db.集合.createIndex({"字段": "hashed"})
```



#### 2. 索引管理

```
//查看索引
db.medical_info.getIndexes()
//查看索引大小
db.medical_info.totalIndexSize()
//索引重建
db.COLLECTION_NAME.reIndex()
//删除索引
db.medical_info.dropIndex("name_1")
db.COLLECTION_NAME.dropIndexes()
注意: _id 对应的索引是删除不了的
```



#### 3. 索引分析 ---explain

explain()接收参数分析：

- queryPlanner：queryPlanner是默认参数。
- executionStats： executionStats会返回执行计划的一些统计信息(有些版本中和allPlansExecution等同)。
- allPlansExecution:allPlansExecution用来获取所有执行计划，结果参数基本与上文相同



executionStats返回参数解析：

![img](http://pcc.huitogo.club/944398946f3531f06d347172d42f1d35)



```
//示例
db.medical_info.find({name:"测试1",age:3}).explain("executionStats")
```

![img](http://pcc.huitogo.club/5b7d384147f7357a58894130f9b88a8f)



#### 4. 索引底层实现原理

MongoDB使用B-树，所有节点都有Data域，只要找到指定索引就可以进行访问，单次查询从结构上来看要快于MySql。



B-树的特点:

- 多路 非二叉树
- 每个节点 既保存数据 又保存索引
- 搜索时 相当于二分查找

![img](http://pcc.huitogo.club/aac9e7e6375b560d180f11e232336e06)



B+ 树的特点:

- 多路非二叉
- 只有叶子节点保存数据
- 搜索时 也相当于二分查找
- 增加了 相邻节点指针

![img](http://pcc.huitogo.club/3ad52f7a45e53a689f27bed1b433640c)



从上面我们可以看出最核心的区别主要有俩，一个是数据的保存位置，一个是相邻节点的指向。就是这俩造成了MongoDB和MySql的差别。

1. B+树相邻接点的指针可以大大增加区间访问性，可使用在范围查询等，而B-树每个节点 key 和data 在一起适合随机读写 ，而区间查找效率很差。
2. B+树更适合外部存储，也就是磁盘存储，使用B-结构的话，每次磁盘预读中的很多数据是用不上的数据。因此，它没能利用好磁盘预读的提供的数据。由于节点内无 data 域，每个节点能索引的范围更大更精确。
3. 注意这个区别相当重要，是基于（1）（2）的，B-树每个节点即保存数据又保存索引树的深度小，所以磁盘IO的次数很少，B+树只有叶子节点保存，较B树而言深度大磁盘IO多，但是区间访问比较好。、