首先总结下MySQL的查询执行顺序

> ```
> (7)     SELECT 
> (8)     DISTINCT <select_list>
> (1)     FROM <left_table>
> (3)     <join_type> JOIN <right_table>
> (2)     ON <join_condition>
> (4)     WHERE <where_condition>
> (5)     GROUP BY <group_by_list>
> (6)     HAVING <having_condition>
> (9)     ORDER BY <order_by_condition>
> (10)    LIMIT <limit_number>
> ```

- 执行顺序即上文中标注的序号
- 在SQL语句的执行过程中，每一步都会产生一个虚拟表（Virtual Table，简称VT），用来保存SQL语句的执行结果。



#### 1. 执行FROM语句

第一步，执行FROM语句。我们首先需要知道最开始从哪个表开始的，这就是FROM告诉我们的。经过FROM语句对两个表执行笛卡尔积，会得到一个虚拟表，暂且叫VT1。总共有——table1的记录条数 * table2的记录条数——条记录，这就是VT1的结果。



#### 2. 执行ON过滤

执行完笛卡尔积以后，接着就进行ON join_condition条件过滤，比如ON a.customer_id = b.customer_id，根据ON中指定的条件，去掉那些不符合条件的数据，得到VT2表。



#### 3. 添加外部行（外联结）

这一步只有在连接类型为OUTER JOIN时才发生，如LEFT OUTER JOIN（左连接）、RIGHT OUTER JOIN（右连接）。大多数时候会省略OUTER关键字。

添加外部行的工作就是在VT2表的基础上添加保留表中被过滤条件过滤掉的数据，非保留表中的数据被赋予NULL值，最后生成虚拟表VT3。

这个稍微难理解举个例子：

```
+-------------+----------+----------+-------------+
| customer_id | city     | order_id | customer_id |
+-------------+----------+----------+-------------+
| baidu       | hangzhou |     NULL | NULL        |
+-------------+----------+----------+-------------+
```

这一行数据要是在上一步ON过滤：ON a.customer_id = b.customer_id中是绝对会被过滤掉的，但是如果我们的外连接是LEFT JOIN,则需要补充这一行数据，成为我们的VT3。回顾一下左连接：

> LEFT JOIN 关键字会从左表 (table_name1) 那里返回所有的行，即使在右表 (table_name2) 中没有匹配的行。

接下来的操作都会在该VT3表上进行。



#### 4. 执行WHERE过滤

对添加外部行得到的VT3进行WHERE过滤，只有符合 where_condition 的记录才会输出到虚拟表VT4中。



#### 5. 执行GROUP BY分组

上面得到的虚拟表还没有经过聚合分组，**GROU BY子句主要是对使用WHERE子句得到的虚拟表进行分组操作。**得到的内容会存入虚拟表VT5中，此时，我们就得到了一个VT5虚拟表，接下来的操作都会在该表上完成。



#### 6. 执行HAVING过滤

这里需要注意的是到目前为止已经有了三种过滤，ON、WHERE和HAVING，三者在执行时间段上是有严格区别的，HAVING子句主要和GROUP BY子句配合使用，对分组得到的VT5虚拟表进行条件过滤，然后得到虚拟表VT6。



#### 7. SELECT列表

从虚拟表VT6中选择出我们需要的内容，生成虚拟表VT7。



#### 8. 执行DISTINCT子句

如果在查询中指定了DISTINCT子句，则会创建一张内存临时表（如果内存放不下，就需要存放在硬盘了）。这张临时表的表结构和上一步产生的虚拟表VT7是一样的，不同的是对进行DISTINCT操作的列增加了一个唯一索引，以此来除重复数据。



#### 9. 执行ORDER BY子句

对虚拟表中的内容按照指定的列进行排序，然后返回一个新的虚拟表，上述结果会存储在VT8中。



#### 10. 执行LIMIT子句

LIMIT子句从上一步得到的VT8虚拟表中选出从指定位置开始的指定行数据。mysql的limit语法如下：

```
LIMIT n,m
```