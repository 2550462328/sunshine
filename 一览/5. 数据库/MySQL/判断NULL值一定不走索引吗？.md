在mysql优化中我们建议将表中的字段设置为非NULL



理由是

- 索引 NULL 列需要额外的空间来保存，所以要占用更多的空间
- 进行比较和计算时要对 NULL 值做特别的处理

还有这里需要澄清的是一个**谣言**，在此之前一直坚信的一点



就是对字段进行!= 、not null 、 <>判断会导致该字段的索引失效！



为什么说这个是谣言呢？

因为你肯定没有自己动手试试，自己用explain语句试一下就知道了



当然我们今天更想知道的是NULL值是什么？对于一个NULL值mysql是怎么存储的？以及查询的时候对于NULL索引是怎么查询的？



#### 1. NULL值是什么？



##### 1.1 MySQL 中的NULL

对MySQL来说，NULL是一个特殊的值。

NULL表示不可知不确定，NULL不与任何值相等（包括其本身）



##### 1.2 NULL占用的长度

NULL在数据库中占用的长度

```
mysql>  select length(NULL), length(''), length('1');
+--------------+------------+-------------+
| length(NULL) | length('') | length('1') |
+--------------+------------+-------------+
| NULL         |          0 |           1 |
+--------------+------------+-------------+
```



*NULL columns require additional space in the row to record whether their values are NULL.*

可以看出空值''的长度是0，是不占用空间的；而的NULL长度是NULL，是需要占用额外空间的，所以在一些开发规范中，建议将数据库字段设置为Not NULL,并且设置默认值''或0。



##### 1.3 对NULL值的比较

IS NULL 判断某个字符是否为空，并不代表空字符或者是0

SQL92标准中说道，!=NULL 条件判断永远返回false,聚合运算永远返回0

当然在数据库中可以使用SET ANSI_NULLS OFF关闭标准模式，但一般不建议这样去做

所以，要判断一个数是否等于NULL只能用 IS NULL 或者 IS NOT NULL 来判断



##### 1.4 SQL对NULL值进行处理

MySQL中专门为我们提供了IFNULL(expr1,expr2)这个函数，让我们可以轻松的处理数据中的NULL

IFNULL有两个参数。 如果第一个参数字段不是NULL，则返回第一个字段的值。 否则，IFNULL函数返回第二个参数的值（默认值）。

```
select IFNULL(status,0) From t_user;
```



##### 1.5 值为NULL 对查询条件的影响

首先需要注意的一点是，MySQL中某一列数据含有NULL，并不一定会造成索引失效。

MySQL可以在含有NULL的列上使用索引

在有NULL值的字段上使用常用的索引，如普通索引、复合索引、全文索引等不会使索引失效。但是在使用空间索引的情况下，该列就必须为 NOT NULL。



##### 1.6 值为NULL对排序的影响

在ORDER BY排序的时候，如果存在NULL值，那么NULL是最小的，ASC正序排序的话，NULL值是在最前面的

如果我们需要在正序排序时，将NULL值放在后边,这里我们就需要巧借IS NULL

```
select * from t_user order by age is null, age; 
或者
select * from t_user order by isnull(name), age; 
# 等价于
select * from (select name, age, (age is null) as isnull from t_user) as foo order by isnull, age; 
```



##### 1.7 NULL和空值区别

NULL也就是在字段中存储NULL值，空值也就是字段中存储空字符('')。

```
mysql>  select length(NULL), length(''), length('1');
+--------------+------------+-------------+
| length(NULL) | length('') | length('1') |
+--------------+------------+-------------+
| NULL         |          0 |           1 |
+--------------+------------+-------------+
1 row in set
```



总结：从上面看出空值('')的长度是0，是不占用空间的；而的NULL长度是NULL，其实它是占用空间的，看下面说明。

*NULL columns require additional space in the row to record whether their values are NULL*



NULL列需要行中的额外空间来记录它们的值是否为NULL。



通俗的讲：**空值就像是一个真空转态杯子，什么都没有，而NULL值就是一个装满空气的杯子，虽然看起来都是一样的，但是有着本质的区别。**



#### 2. NULL值是怎么在记录中存储的？

在MySQL中，每一条记录都有它固定的格式，我们以InnoDB存储引擎的Compact行格式为例，来看一下NULL值是怎样存储的。在Compact行格式下，一条记录是由下边这几个部分构成的：

![img](http://pcc.huitogo.club/28e57fb5f63a294042e2ac74ff891eac)



我们举例说明



首先创建表record_format_demo

```
1. CREATE TABLE record_format_demo ( 

2.   c1 VARCHAR(10), 

3.   c2 VARCHAR(10) NOT NULL, 

4.   c3 CHAR(10), 

5.   c4 VARCHAR(10) 

6. ) CHARSET=ascii ROW_FORMAT=COMPACT; 
```



是的，没错，我们需要关注的就是每行记录的额外信息**NULL值列表**



存储NULL值的过程如下：

1）首先统计表中允许存储NULL的列有哪些。

比方说表record_format_demo的3个列c1、c3、c4都是允许存储NULL值的，而c2列是被NOT NULL修饰，不允许存储NULL值。



2）如果表中没有允许存储NULL的列，则NULL值列表也不存在了，否则将每个允许存储NULL的列对应一个二进制位，二进制位按照列的顺序逆序排列

因为表record_format_demo有3个值允许为NULL的列，所以这3个列和二进制位的对应关系就是这样：

![img](http://pcc.huitogo.club/3d17429f65a094c471b46b3be209ae59)



再一次强调，二进制位按照列的顺序逆序排列，所以第一个列c1和最后一个二进制位对应。



3）设计InnoDB的大牛规定NULL值列表必须用整数个字节的位表示，如果使用的二进制位个数不是整数个字节，则在字节的高位补0。

表record_format_demo只有3个值允许为NULL的列，对应3个二进制位，不足一个字节，所以在字节的高位补0，效果就是这样

![img](http://pcc.huitogo.club/5d5ff666e4d14d8ad0888fba00819615)



假设我们现在向record_format_demo表中插入一条记录：

```
1. INSERT INTO record_format_demo(c1, c2, c3, c4) 

2.   VALUES('eeee', 'fff', NULL, NULL); 
```



这条记录的c1、c3、c4这3个列中c3和c4的值都为NULL，所以这3个列对应的二进制位的情况就是：

![img](http://pcc.huitogo.club/a038ab18b43440f53bc10a04c65fb92e)

二进制位表示的意义如下：

- 二进制位的值为1时，代表该列的值为NULL。
- 二进制位的值为0时，代表该列的值不为NULL。

所以这记录的NULL值列表用十六进制表示就是：0x06。

所以存储NULL需要额外的空间就是因为需要存储这个NULL值列表



因此建议字段设置为非NULL，如果那个字段确实没有值，可以用’ ’填充

![img](http://pcc.huitogo.club/4b70ef8b77a14b62b07c1f5e91536106)



因为’ ’是不会占据空间的。



#### 3. 使不使用索引的依据到底是什么？

答案很简单：**成本**。



对于使用二级索引进行查询来说，成本组成主要有两个方面：

1. 读取二级索引记录的成本
2. 将二级索引记录执行回表操作，也就是到聚簇索引中找到完整的用户记录的操作所付出的成本。



很显然，**要扫描的二级索引记录条数越多，那么需要执行的回表操作的次数也就越多**，达到了某个比例时，使用二级索引执行查询的成本也就超过了全表扫描的成本（举一个极端的例子，比方说要扫描的全部的二级索引记录，那就要对每条记录执行一遍回表操作，自然不如直接扫描聚簇索引来的快）。



所以MySQL优化器在真正执行查询之前，对于每个可能使用到的索引来说，都会预先计算一下需要扫描的二级索引记录的数量，比方说对于下边这个查询：

```
 // 查询空语句
 
1. SELECT * FROM s1 WHERE key1 IS NULL; 
```



优化器会分析出此查询只需要查找key1值为NULL的记录，然后访问一下二级索引idx_key1，看一下值为NULL的记录有多少（如果符合条件的二级索引记录数量较少，那么统计结果是精确的，如果太多的话，会采用一定的手段计算一个模糊的值，当然算法也比较麻烦，我们就不展开说了），这种在查询真正执行前优化器就率先访问索引来计算需要扫描的索引记录数量的方式称之为index dive。



当然，对于某些查询，比方说WHERE子句中有IN条件，并且IN条件中包含许多参数的话，



比方说这样：

```
 // in查询语句

1. SELECT * FROM s1 WHERE key1 IN ('a', 'b', 'c', ... , 'zzzzzzz'); 
```



这样的话需要统计的key1值所在的区间就太多了，这样就不能采用index dive的方式去真正的访问二级索引idx_key1，而是需要采用之前在背地里产生的一些统计数据去估算匹配的二级索引记录有多少条（很显然根据统计数据去估算记录条数比index dive的方式精确性差了很多）。



反正不论采用index dive还是依据统计数据估算，最终要得到一个需要扫描的二级索引记录条数，**如果这个条数占整个记录条数的比例特别大，那么就趋向于使用全表扫描执行查询，否则趋向于使用这个索引执行查询**。



理解了这个也就好理解为什么在WHERE子句中出现IS NULL、IS NOT NULL、!=这些条件仍然可以使用索引，本质上都是优化器去计算一下对应的二级索引数量占所有记录数量的比值而已。