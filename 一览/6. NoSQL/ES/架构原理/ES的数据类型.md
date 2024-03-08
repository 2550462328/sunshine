#### 1. 索引（Index）

ES 中存储数据的**基本单位是索引**，比如说你现在要在 ES 中存储一些订单数据，你就应该在 ES 中创建一个索引 `order_idx` ，所有的订单数据就都写到这个索引里面去，一个索引差不多就是相当于是 mysql 里的一个数据库。

```
index -> type -> mapping -> document -> field
```



**elasticsearch类比mysql**

![img](http://pcc.huitogo.club/d0528357010d8b1bb37ab69d10f53a2b)



声明索引结构如下：

![img](http://pcc.huitogo.club/5ae3feda1a337446feb8c3c3d293c740)

1）索引是文档的容器，是一类文档的结合

Index体现了逻辑空间的概念：每个索引都有自己的Mapping定义，用于定义包含的文档的字段名和字段类型

Shard体现了物理空间的概念：索引中的数据分散在Shard上



2）索引的Mapping与Settings

- Mapping定义文档字段的类型
- Setting定义不同的数据分布



#### 2. Type

index 相当于 mysql 数据库。而 type 没法跟 mysql 里去对比，一个 index 里可以有多个 type，每个 type 的字段都是差不多的，但是有一些略微的差别。假设有一个 index，是订单 index，里面专门是放订单数据的。就好比说你在 mysql 中建表，有些订单是实物商品的订单，比如一件衣服、一双鞋子；有些订单是虚拟商品的订单，比如游戏点卡，话费充值。就两种订单大部分字段是一样的，但是少部分字段可能有略微的一些差别。

所以就会在订单 index 里，建两个 type，一个是实物商品订单 type，一个是虚拟商品订单 type，这两个 type 大部分字段是一样的，少部分字段是不一样的。

很多情况下，一个 index 里可能就一个 type，但是确实如果说是一个 index 里有多个 type 的情况，你可以认为 index 是一个类别的表，具体的每个 type 代表了 mysql 中的一个表。

![es-index-type-mapping-document-field](https://doocs.gitee.io/advanced-java/docs/high-concurrency/images/es-index-type-mapping-document-field.png)



**注意**： `mapping types` 这个概念在 ElasticSearch 7. X 已被完全移除，详细说明可以参考[官方文档](https://github.com/elastic/elasticsearch/blob/6.5/docs/reference/mapping/removal_of_types.asciidoc)



#### 3.  Mapping

每个 type 有一个 mapping，如果你认为一个 type 是具体的一个表，index 就代表多个 type 同属于的一个类型，而 mapping 就是这个 type 的**表结构定义**。



1）Mapping 类似数据库 中的schema的定义，作用如下

- 定义索引中的字段的名称
- 定义字段的数据类型，例如字符串，数字，布尔...
- 字段，倒排索引的相关配置， (Analyzed or Not Analyzed, Analyzer)



2）Mapping 会把JSON 文档映射成Lucene 所需要的扁平格式



3）一个索引对应一个Mapping

Mapping中可以设置字段Type

![img](http://pcc.huitogo.club/1496bf1e76f954e686dcffa2acf683d1)



其中四种不同级别的Index Options配置，可以控制倒排索引记录的内容

-  docs - 记录doc id

-  freqs - 记录doc id和term frequencies

-  positions -记录doc id / term frequencies / term position

-  offsets - doc id / term frequencies / term posistion / character offsets

  

Text类型默认记录postions，其他默认为docs

7.0 开始，不需要在Mapping定义中指定type信息



#### 4. 文档（Document）

1）Elasticsearch是面向文档的，文档是所有可搜索数据的最小单位，比如

- 日志文件中的日志项
- 一本电影的具体信息/一张唱片的详细信息
- MP3播放器里的一首歌/一篇PDF文档中的具体内容



2）文档会被序列化成JSON格式，保存在Elasticsearch中

- JSON对象由字段组成,
- 每个字段都有对应的字段类型(字符串/数值/布尔/日期/二进制/范围类型)



3）每个文档都有一一个Unique ID

你可以自己指定ID

或者通过Elasticsearch自动生成



#### 4. 文档元数据

![img](http://pcc.huitogo.club/7d238d48a0155ff3bb81c0aa4c8e8d86)



其中字段说明：

- _index: 文档所属的索引名

- _type: 文档所属的类型名

- _id: 文档唯一Id

- _source: 文档的原始Json数据

- _all: 整合所有字段内容到该字段，已被废除

- _version: 文档的版本信息

- _score: 搜索相关性评分



#### 5. 数据类型

##### 5.1 简单类型

- Text / Keyword:字符型最常用的，text会默认分词，keyword当做整体处理
- integer:整型
- long:长整型
- float:浮点型
- double:双字节型
- boolean：布尔型



##### 5.2 复杂类型

- array：数组型

- “lists”:{{“name”:”…”},{“name”:”…”}}

- object:对象类型

  ```
  “author”:{“type”:”object”,”perperites”:{“name”:{“type”:”string”}}}
  ```

  











