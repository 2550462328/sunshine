**通过 explain sql语句 分析查询语句**

可以通过explain sql语句来了解当前语句的缺陷来进一步进行调优

执行完explain sql语句后展示如下：

![img](http://pcc.huitogo.club/d74397a8b140f204a3d3c437faed8000)



##### 1. id

决定着语句的执行顺序，都为1的时候就顺序执行，**id数值越大执行优先级越高**。

理解上就是嵌在内部的sql查询语句先执行。



##### 2. select_type

当前查询语句的查询类型，有六个类型

- simple

  简单查询，查询中不包含子查询或者union等任何复杂查询。

- primary

  查询中若包含任何复杂的子查询，则最外层被标记为primary，俗称是鸡蛋壳

- subquery

  在select或where列表包含了子查询

![img](http://pcc.huitogo.club/692f156a7b1d0663347a2c1c56d19930)

- derived

  在from列表中包含的子查询被标记为derived（衍生），mysql会递归执行这些子查询，把结果放在临时表里（临时表会增加系统负担，但有时不得不用）

![img](http://pcc.huitogo.club/58d14bcdbeeb2b4f5ee0823eee204322)

- union

  若第二个select出现在union之后，则被标记为union；若union包含在from子句的子查询中，外层select将被标记为：derived

- union result

  两种union结果的合并

![img](http://pcc.huitogo.club/ebe6eb6df639b876396f947a5e090325)



##### 3. table

当前sql查询语句查询的表名



##### 4. type

查询的范围，也可以代指查询的效率

从最好到最差依次是：

> system > const > eq_ref > ref > range > index > all(全表扫描)



###### 4.1 system

表只要一行记录（等于系统表），这是const类型的特例，平时不会出现，这个也可以忽略不计。



###### 4.2 const

表示通过索引一次就找到了，const用于primary key 或者 unique key，因为只匹配一行数据，所以很快，如将主键置于where列表中，mysql就能将该查询转换为一个常量。

![img](http://pcc.huitogo.club/341a6ad693a66cc64362cb02c4f46368)



###### 4.3 eq_ref

也是用于唯一索引，primary key或者unique key，跟const不同在于，这个唯一索引用在了“外键“

![img](http://pcc.huitogo.club/1701f14baf65ef8e7c382181092d7201)



###### 4.4 ref

非唯一性索引扫描，返回匹配某个单独值的所有行，本质上也是一种索引访问，它返回所有匹配某个单独值的行，然而它可能会找到多个符合条件的行，所以他应该属于查找和扫描的混合体



###### 4.5 range

只检索给定范围的行，使用一个索引来选择行，key列显示使用了哪个索引，一般就是你的where语句中出现了between、<、>、in等的查询(mysql5.7支持in走索引)，这种范围扫描索引扫描比全表扫描好，因为它至于要开始索引的某一点，而结束语另一点，不用扫描全部索引

![img](http://pcc.huitogo.club/76a4fd8ea1cc31d6a6d7e0886636776a)



###### 4.6 index

full index scan(全索引扫描)，index与all区别为index类型只遍历索引树，这通常比all块，因为索引文件通常比数据文件小。（也就是说虽然all和index都是读全表，单index是从索引中读取的，而all是从硬盘中读的）

![img](http://pcc.huitogo.club/fdef1064000c083cd0468859cf324ea5)



###### 4.7 all

全表扫描，不具备任何优化点



**SQL 性能优化的目标：至少要达到 range 级别，要求是 ref 级别，如果可以是consts最好。**



##### 5. possible_keys

显示**可能**应用在这张表中的索引，一个或多个。查询涉及到的字段上若存在索引，则该索引将被列出，但**不一定被查询实际使用**



##### 6. key

实际上使用到的索引，如果为null，则没有使用索引

查询中若使用了(覆盖索引)，则该索引仅出现在key列表中

**覆盖索引就是查询的字段都在索引表里**，这样查询数据直接查索引表里面的值就可以了，不用去硬盘里面查表数据。

覆盖索引是组合索引的特殊情况，就是使用组合索引查询出来的列**刚好是组合索引的列或者从左边开始的部分列**。



##### 7. key_len

表示索引中使用的字节数，可通过该列计算查询中使用的索引的长度，在不损失精确性的情况下，长度越短越好。

key_len显示的值为索引字段的最大可能长度，并非实际使用长度，即**key_len是根据表定义计算而得，不是通过表内检索出的**。



##### 8. ref

显示索引的哪一列被使用了，如果可能的话，是一个常数。哪些列或常量被用于查找索引列上的值



##### 9. rows

根据表统计信息及索引选用情况，大致估算出找到所需的记录所需要读取的行数(越少越好)，每张表有多少行被优化器查询



##### 10. extra

包含不适合在其他列显示但是很重要的额外信息，有4种情况

###### 10.1 using filrsort（危险）

说明mysql会对数据使用一个外部的索引排序，而不是按照表内的索引顺序进行读取。mysql中**无法利用索引完成的排序**操作称为"文件排序"，一旦出现这种情况很危险。

就是**order by后面的字段不在索引字段范围内**



###### 10.2 using index

使用到了索引、这种情况是好事。表示响应的select操作中使用了覆盖索引（cocering index），避免访问了表的数据行，效率不错！如果同时出现了using where，表名索引被用来执行索引键值的查找；如果没有同时出现using where，表名索引用来读取数据而非执行查找动作。



###### 10.3 using where

使用了where条件



###### 10.4 using temporary

使用了临时表保存中间结果。mysql在对查询结果排序时使用临时表。常见于排序order by和分组查询 group by。（group by 最好与索引的字段、顺序一致）



下面这种情况下，主键是sysId，组合索引顺序是userType,platId,userName,userPassword

![img](http://pcc.huitogo.club/76b59426ac96854868a74d875e7ef774)