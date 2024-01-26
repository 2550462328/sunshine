JSON 数据类型是在mysql5.7版本后新增的，同 TEXT，BLOB 字段一样，JSON 字段不允许直接创建索引。即使支持，实际意义也不大，因为我们一般是基于文档中的元素进行查询，很少会基于整个 JSON 文档。

基于此问题，在MySQL 8.0.17及以后的版本中，InnoDB存储引擎支持JSON数组上的**多值索引**。除此之外还可以通过MySQL 5.7 引入的**虚拟列**，然后在虚拟列当中使用索引。



#### 1. 虚拟列

- **InnoDB支持在虚拟生成的列上建立二级索引**。不支持其他索引类型（主键索引）。在虚拟列上定义的二级索引有时也称为“**虚拟索引**”。
- **二级索引可以在一个或多个虚拟列上创建**，也可以在虚拟列与常规列或存储生成列的**组合上创建**。包含虚拟列的二级索引**可以定义为UNIQUE**。
- 当在虚拟列上使用辅助索引时，由于在INSERT和UPDATE操作期间在辅助索引（辅助又叫二级索引）记录中实现虚拟列值时执行计算，因此需要考虑额外的写成本。即使有额外的写成本，虚拟列上的二级索引也可能比生成的存储列更可取，生成的存储列在集群索引中具体化，从而导致需要更多磁盘空间和内存的更大的表。如果没有在虚拟列上定义二级索引，则会产生额外的读取成本，因为每次检查列的行时都必须计算虚拟列值。



**语法**：ALTER TABLE 表名称 add column 虚拟列名称 虚拟列类型 GENERATED ALWAYS as (表达式) [VIRTUAL | STORED];



**MySQL 在处理 虚拟列存储问题的时候有两种方式：**

- **VIRTUAL**（默认）：不存储列值，在读取表的时候自动计算并返回，不消耗任何存储，这种存储方式仅 InnoDB 支持设置索引。
- **STORED**：在插入或更新时计算存储列值，存储的虚拟列需要存储空间，并且 MyISAM 也可以设置索引。
- 

**创建虚拟列**可以在创建表的时候指定也可以在创建表过后指定。

1）创建表

```
mysql> CREATE TABLE jemp (
    ->     c JSON,
    ->     g INT GENERATED ALWAYS AS (c->"$.id"),
    ->     INDEX i (g)
    -> );
```



**2）修改表**

```
ALTER TABLE gpt_track_event ADD COLUMN json_reportId VARCHAR(32) GENERATED ALWAYS AS (event_info->'$.reportId') VIRTUAL;
CREATE INDEX idx_json_reportId ON gpt_track_event(json_reportId);
```



#### 2. 多值索引

多值的索引从MySQL 8.0.17开始，InnoDB支持多值索引。多值索引是在存储值数组的列上定义的二级索引。“普通”索引对每个数据记录有一个索引记录(1:1)。一个多值索引对于一个数据记录(N:1)可以有多个索引记录。多值索引用于索引JSON数组。



例如，在下面的JSON文档中，我们要对zipcode添加一个索引：

```
{
    "user":"Bob",
    "user_id":31,
    "zipcode":[94477,94536]
}
```



**三种创建多值索引的方式： CREATE TABLE, ALTER TABLE, or CREATE INDEX**

**1）CREATE TABLE**

```
CREATE TABLE customers (
    id BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
    modified DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    custinfo JSON,
    INDEX zips( (CAST(custinfo->'$.zipcode' AS UNSIGNED ARRAY)) )
);
```



**2）ALTER TABLE**

```
CREATE TABLE customers (
id BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
modified DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
custinfo JSON
);
    
ALTER TABLE customers ADD INDEX zips( (CAST(custinfo->'$.zipcode' AS UNSIGNED ARRAY)) );
```



**3）CREATE INDEX**

```
CREATE INDEX zips ON customers ( (CAST(custinfo->'$.zipcode' AS UNSIGNED ARRAY)) );
```



想要**多值索引生效**的条件是 where条件下使用了以下三个函数：

- MEMBER OF()：查看数组是否有某个元素，如果有则该函数返回 1，否则返回 0。

​      语法：元素 value MEMBER OF(json_array)

- JSON_CONTAINS()：该函数用于检验指定 JSON 文档是否包含在目标 JSON 文档中，或者是否在目标文档的指定路径上找到指定元素（如果提供了 path参数）。如果指定 JSON 文档包含在目标 JSON 文档中，该函数返回 1，否则返回 0。

​       语法：JSON_CONTAINS(target, candidate[, path])

- JSON_OVERLAPS()：该函数用于比较两个 JSON 文档。如果两个文档具有共同的键值对（key-value）或数组元素（不要求全部一样，只要一个键值对一样就可以），则返回 1，否则返回 0。

​       语法：JSON_OVERLAPS(json_doc1, json_doc2)

```
EXPLAIN SELECT * FROM customers WHERE 94507 MEMBER OF(custinfo->'$.zipcode');

EXPLAIN SELECT * FROM customers WHERE JSON_CONTAINS(custinfo->'$.zipcode', CAST('[94507,94582]' AS JSON));

EXPLAIN SELECT * FROM customers WHERE JSON_OVERLAPS(custinfo->'$.zipcode', CAST('[94507,94582]' AS JSON));
```



使用的时候**需要注意**的：

- 多值索引可以定义为唯一键，不能作为主键，和外键。
- 可以作为组合索引使用
- 不支持utf8mb4编码配合utf8mb4_0900_as_cs排序规则使用，不支持默认的二进制排序规则和字符集。
- 多值索引不能是覆盖索引。
- 不能为多值索引定义索引前缀。



T1：**覆盖索引** - 索引是高效找到行的一个方法，当能通过检索索引就可以读取想要的数据，那就不需要再到数据表中读取行了。如果一个索引包含了（或覆盖了）满足查询语句中字段与条件的数据就叫 做覆盖索引。



T2：**前缀索引** - 所谓前缀索引说白了就是对文本的前几个字符建立索引（具体是几个字符在建立索引时指定），这样建立起来的索引更小，所以查询更快。这有点类似于 Oracle 中对字段使用 Left 函数来建立函数索引，只不过 MySQL 的这个前缀索引在查询时是内部自动完成匹配的，并不需要使用 Left 函数。

那么为什么不对整个字段建立索引呢？一般来说使用前缀索引，可能都是因为整个字段的数据量太大，没有必要针对整个字段建立索引，前缀索引仅仅是选择一个字段的部分字符作为索引，这样一方面可以节约索引空间，另一方面则可以提高索引效率，当然很明显，这种方式也会降低索引的选择性。