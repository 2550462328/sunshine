#### 1. binlog是什么？

`bin log`是逻辑日志，记录的是执行语句的逻辑，和`redis`的`AOP`日志类似，会按顺序记录所有涉及更新数据的逻辑操作。



**主要作用：**

- 数据恢复：`MySQL`可以通过`bin log`恢复某一时刻的误操作的数据，是`DBA`常打交道的日志。
- 数据复制：`MySQL`的数据备份、集群高可用、读写分离都是基于`bin log`的重放实现的。

![img](https://pcc.huitogo.club/z0/v2-0bd2110f68d8ccf2b92a38569fcfde19_720w.webp)



#### 2. binlog的记录格式

`binlog`日志有三种格式：`statement`、`row`和`mixed`，对比如下：

| 格式      | 含义                                                         | 优点                                                         | 缺点                                                         |
| --------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| statement | 基于SQL语句的复制，记录的是更新数据操作的SQL语句，这些语句同步时会被其他节点执行，如update T set time=NOW() where id = 1; | 不需要记录数据的变化，减少了bin log文件大小，减少IO负担。    | SQL中包含了每次执行结果不一致的函数、触发器时，同步数据时会造成不一致。 |
| row       | 基于行的复制，5.1.5版本支持的格式，记录了数据被更改的具体值，如update T set time=1687843346000 where id = 1; | 日志会清晰记录每条数据被修改的详细情况，保证了数据的一致性。 | 每条数据的更改被详细记录，如整表删除，alter表等操作涉及的数据行都会记录，ROW格式会产生大量日志。 |
| mixed     | 混合模式，5.1.8版本开始，以上两种格式的混合版，对于DDL只对SQL语句进行记录，对DML操作则会进行判断，如果判断会造成主从不一致，就会采用row格式记录，反之则用statement格式记录。 | 既节省空间，又提高数据库性能，保证数据同步时的一致性。       | 无法对误操作数据进行单独恢复。                               |

新版本的 MySQL 中对 row level 模式也被做了优化，并不是所有的修改都会以 row level 来记录。

- 像遇到表结构变更的时候就会以 Statement 模式来记录。
- 至于 Update 或者 Delete 等修改数据的语句，还是会记录所有行的变更，即使用 Row 模式。



**怎么选择？**

- Statement 可能占用空间会相对小一些，传送到 slave 的时间可能也短，但是没有 Row 模式的可靠。
- Row 模式在操作多行数据时更占用空间，但是可靠。

所以，这是在占用空间和可靠之间的选择。互联网公司，使用 MySQL 的功能相对少，基本不使用存储过程、触发器、函数的功能，选择默认的语句模式，Statement Level（默认）即可。



**如何在线正确清理 MySQL binlog？**

MySQL 中的 binlog 日志记录了数据中的数据变动，便于对数据的基于时间点和基于位置的恢复。但日志文件的大小会越来越大，占用大量的磁盘空间，因此需要定时清理一部分日志信息。

```
# 首先查看主从库正在使用的binlog文件名称
show master(slave) status

# 删除之前一定要备份
purge master logs before'2017-09-01 00:00:00'; # 删除指定时间前的日志
purge master logs to'mysql-bin.000001'; # 删除指定的日志文件

# 自动删除：通过设置binlog的过期时间让系统自动删除日志
show variables like 'expire_logs_days'; # 查看过期时间
set global expire_logs_days = 30; # 设置过期时间
```