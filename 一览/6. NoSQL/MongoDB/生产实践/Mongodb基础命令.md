#### 1. 基础操作

```
查看数据库
show dbs;
切换数据库 如果没有对应的数据库则创建
use 数据库名;
创建集合
db.createCollection("集合名")
查看集合
show tables;
show collections;
删除集合
db.集合名.drop();
删除当前数据库
db.dropDatabase();
```



#### 2. 集合操作

##### 2.1 新增

```
单文档新增
db.集合名.insert({})
多文档新增
db.集合名.insert([{},{}....])

示例：
//单文档插入
db.mongo_test.insert([{name:"张三1",age:26}])
//多文档插入
db.mongo_test.insert([{name:"张三2",age:27},{name:"张三2",age:28},{name:"张三2",age:29},{name:"张三2",age:30}])
```



##### 2.2 查询

```
db.集合名.find({条件})
```

![img](http://pcc.huitogo.club/0217d4d046830d518ee0e101ddbc7ddb)



示例：

```
//查询
db.medical_info.find({})
//name=测试10
db.medical_info.find({name:"测试10"})
db.medical_info.find({name:{$eq: "测试10"}})
//age>=60
db.medical_info.find({age:{$gte: 60}}).project({name:1,age:1})
//逻辑运算
//and name=测试10 并且 年龄=47
db.medical_info.find({name:"测试10",age:47}).project({name:1,age:1})
db.medical_info.find({$and: [{name:"测试10",age:47}]}).project({name:1,age:1})
db.medical_info.find({$and: [{name:"测试10"},{age:47}]}).project({name:1,age:1})
//or name=测试10 或者 年龄=47
db.medical_info.find({$or: [{name:"测试10"},{age:47}]}).project({name:1,age:1})
//or name=测试10，年龄=47 或者 name=测试11，年龄=23
db.medical_info.find({$or: [{name:"测试10",age:47},{name:"测试
11",age:23}]}).project({name:1,age:1})
//not 年龄不小于60岁的
db.medical_info.find({age:{$not: {$lt: 60}}}).project({name:1,age:1})
//not in name=测试10 年龄not in [47,66]
db.medical_info.find({name:"测试10",age:{$in: [47,66]}}).project({name:1,age:1})
db.medical_info.find({age:{$not: {$in: [47,66]}}}).project({name:1,age:1})
db.medical_info.find({$and: [{"name":"测试10"},{age:{$not: {$in:[47,66]}}}]}).project({name:1,age:1})
//分页查询
//每页10条，查询第二页数据
db.medical_info.find({name:"测试10"}).sort({ _id:-1}).skip(10).limit(10)
```



##### 2.3 更新

```
db.集合名.update({query},{update},{ multi: false, upsert: false})

query: update的查询条件，类似sql update查询内where后面的
update: update的对象和一些更新的操作符（如$set,$inc...）等
multi: 可选，MongoDB 默认是false,只更新找到的第一条记录，如果这个参数为true,就把按条件查
出来多条记录全部更新
upsert: 可选，这个参数的意思是，如果不存在update的记录，是否插入objNew,true为插入，默认
是false，不插入
操作符：
    $set ：设置字段值，无字段则新增字段
    $unset :删除指定字段
    $inc：对修改的值进行自增
    $rename: 修改字段名
```



示例：

```
//更新属性值 $set
db.medical_info.find({name: "测试1"}).project({name:1,mainSuit:1})
//1、mainSuit变更，单条更新
db.medical_info.update({name:"测试1"},{$set: {mainSuit:"头晕两周"}})
//2、mainSuit变更，多条更新
db.medical_info.update({name:"测试1"},{$set: {mainSuit:"头晕两周"}},{multi:true})
//3、未匹配到更新数据，插入
db.medical_info.update({name:"测试111"},{$set: {mainSuit:"头晕三周"}},
{upsert:true})
//属性值 $inc 自增\自减
db.medical_info.update({name:"测试1",age:3},{$inc: {certificateType:1}})
//修改字段名 $rename
db.medical_info.update({name:"测试1",age:3},{$rename: {mainSuit:"mainSuit1"}})
db.medical_info.find({name:"测试1",age:3})
//删除字段 $unset
db.medical_info.update({name:"测试1",age:3},{$unset: {mainSuit1: 1}})
//添加字段 $set
db.medical_info.update({name:"测试1",age:3},{$set: {mainSuit:"头晕三周"}})
```



##### 2.4 删除

```
db.medical_info.remove({查询条件},
{justOne: true})
justOne : （可选）如果设为 true 或 1，则只删除一个文档，如果不设置该参数，或使用默认值
false，则删除所有匹配条件的文档。
//删除
db.medical_info.remove({_id: ObjectId("622aef2bfe8c28017d864ce4")}, {justOne:
false}
```



##### 2.5 聚合操作

**单目的聚合操作**

单目的聚合命令常用的有：count() 和 distinct()



示例：

```
db.medical_info.count()
db.medical_info.distinct("name")
```



#### 3. 聚合管道(Aggregation Pipeline)

聚合是MongoDB的高级查询语言， 聚合管道将MongoDB文档在一个管道处理完毕后将结果传递给下一个管道处理。管道操作是可以重复的。

![img](http://pcc.huitogo.club/8da7d3a36712ee1d765be39b43cfee4c)



聚合框架中常用的几个操作符：

- $group：将集合中的文档分组，可用于统计结果。
- $project：修改输入文档的结构。可以用来重命名、增加或删除字段，也可以用于创建计算结果以及嵌套文档。
- $match：用于过滤数据，只输出符合条件的文档。$match使用MongoDB的标准查询操作。
- $limit：用来限制MongoDB聚合管道返回的文档数。
- $skip：在聚合管道中跳过指定数量的文档，并返回余下的文档。
- $sort：将输入文档排序后输出。
- $geoNear：输出接近某一地理位置的有序文档
- $unwind: 将数组元素拆分为独立字段
- $lookup: 多表关联（3.2版本新增），相当于mysql join操作



##### 3.1 常见表达式

示例：

```
//获取重名人的年龄（类似于mysql的group_concat）
db.medical_info.aggregate({$group: { _id: "$name",ages:{$push: "$age"}}})
//将所有列都添加到数组中用$push:$$ROOT
db.medical_info.aggregate({$match: {name:{$in: ["测试1","测试2","测试3","测试4","测
试5"]}}},{$group: { _id: "$name",ages:{$push: "$$ROOT"}}})
//统计20岁以上的重名人数并且数量大于20的
db.medical_info.aggregate(
{$match: {age:{$gte: 20}}},
{$group: { _id: "$name", count:{$sum: 1}}},
{$match: {count:{$gte:20}}},
{$project: {cnt:"$count"}}
)
//将diagList数组拆开
db.medical_info.aggregate({$match: {name:"测试99",age:5}},{$project:{name:1,age:1,diagList:1}},{$unwind: "$diagList"})
//$lookup
$lookup: {
from: "<collection to join>",
localField: "<field from the input documents>",
foreignField: "<field from the documents of the from collection>",
as: "<output array field>"
}
db.medical_info.aggregate(
{$match: {name:"测试1",age:8}},
{$lookup: {
from: "medical_info_extend",
localField: "name",
foreignField: "name",
as: "extend"
}})
```



##### 3.2 MapReduce编程模型

Pipeline查询速度快于MapReduce，但是MapReduce的强大之处在于能够在多台Server上并行执行复杂的聚合逻辑。MongoDB不允许Pipeline的单个聚合操作占用过多的系统内存，如果一个聚合操作消耗20%以上的内存，那么MongoDB直接停止操作，并向客户端输出错误消息。



MapReduce是一种计算模型，简单的说就是将大批量的工作（数据）分解（MAP）执行，然后再将结果合并成最终结果（REDUCE）。

```
db.medical_info.mapReduce(
function () {emit(this.cust_id, this.amount)}, //mapFunction
(key, values)=>{return Array.sum(values)},//reduceFunction
{
query:{ status: "A" },
out: { "inline":1 }
})
```



参数说明：

- map：是JavaScript 函数，负责将每一个输入文档转换为零或多个文档，生成键值对序列,作为reduce 函数参数

```
db.medical_info.mapReduce(
function () {emit(this.cust_id, this.amount)}, //mapFunction
(key, values)=>{return Array.sum(values)},//reduceFunction
{
query:{ status: "A" },
out: { "inline":1 }
})
```



- reduce：是JavaScript 函数，对map操作的输出做合并的化简的操作（将key-value变成keyvalues，也就是value数组变成一个单一的值value）
- out：统计结果存放集合
- query： 一个筛选条件，只有满足条件的文档才会调用map函数。
- sort： 和limit结合的sort排序参数（也是在发往map函数前给文档排序），可以优化分组机制
- limit： 发往map函数的文档数量的上限（要是没有limit，单独使用sort的用处不大）
- finalize：可以对reduce输出结果再一次修改
- verbose：是否包括结果信息中的时间信息，默认为false



示例：

```
//统计同名人的平均年龄
db.medical_info.mapReduce(
function () {emit(this.name, this.age)}, //mapFunction
function(key, values) {return Array.avg(values)},//reduceFunction
{
query:{ name: {$in: ["测试1","测试2","测试3","测试4","测试5"]}},
out: "avg_age"
})
```