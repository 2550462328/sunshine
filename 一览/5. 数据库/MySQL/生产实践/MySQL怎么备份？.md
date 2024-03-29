#### 1. 数据的备份类型

- 【常用】完全备份

  > 这是大多数人常用的方式，它可以备份整个数据库，包含用户表、系统表、索引、视图和存储过程等所有数据库对象。但它需要花费更多的时间和空间，所以，一般推荐一周做一次完全备份。

- 增量备份

  > 它是只备份数据库一部分的另一种方法，它不使用事务日志，相反，它使用整个数据库的一种新映象。它比最初的完全备份小，因为它只包含自上次完全备份以来所改变的数据库。它的优点是存储和恢复速度快。推荐每天做一次差异备份。

- 【常用】事务日志备份

  > 事务日志是一个单独的文件，它记录数据库的改变，备份的时候只需要复制自上次备份以来对数据库所做的改变，所以只需要很少的时间。为了使数据库具有鲁棒性，推荐每小时甚至更频繁的备份事务日志。

- 文件备份

  > 数据库可以由硬盘上的许多文件构成。如果这个数据库非常大，并且一个晚上也不能将它备份完，那么可以使用文件备份每晚备份数据库的一部分。由于一般情况下数据库不会大到必须使用多个文件存储，所以这种备份不是很常用。



#### 2. 备份数据的类型

- 热备份
- 温备份
- 冷备份



#### 3. 备份工具

- cp
- mysqldump
- xtrabackup
- lvm2 快照



#### 4. MySQL 几种备份方式

MySQL 一般有 3 种备份方式。

1）逻辑备份

使用 MySQL 自带的 mysqldump 工具进行备份。备份成sql文件形式。

- 优点：最大好处是能够与正在运行的 MySQL 自动协同工作，在运行期间可以确保备份是当时的点，它会自动将对应操作的表锁定，不允许其他用户修改(只能访问)。可能会阻止修改操作。SQL 文件通用方便移植。
- 缺点：备份的速度比较慢。如果是数据量很多的时候，就很耗时间。如果数据库服务器处在提供给用户服务状态，在这段长时间操作过程中，意味着要锁定表(一般是读锁定，只能读不能写入数据)，那么服务就会影响的。

2）物理备份

> 因为现在主流是 InnoDB ，所以基本不再考虑这种方式。

直接拷贝只适用于 MyISAM 类型的表。这种类型的表是与机器独立的。但实际情况是，你设计数据库的时候不可能全部使用 MyISAM 类型表。你也不可能因为 MyISAM 类型表与机器独立，方便移植，于是就选择这种表，这并不是选择它的理由。

- 缺点：你不能去操作正在运行的 MySQL 服务器(在拷贝的过程中有用户通过应用程序访问更新数据，这样就无法备份当时的数据)，可能无法移植到其他机器上去。

3）双机热备份

当数据量太大的时候备份是一个很大的问题，MySQL 数据库提供了一种主从备份的机制，也就是双机热备。

- 优点：适合数据量大的时候。现在明白了，大的互联网公司对于 MySQL 数据备份，都是采用热机备份。搭建多台数据库服务器，进行主从复制。



#### 5.  **数据库不能停机，请问如何备份?** 

可以使用逻辑备份和双机热备份。

- 完全备份：完整备份一般一段时间进行一次，且在网站访问量最小的时候，这样常借助批处理文件定时备份。主要是写一个批处理文件在里面写上处理程序的绝对路径然后把要处理的东西写在后面，即完全备份数据库。
- 增量备份：对 ddl 和 dml 语句进行二进制备份。且 5.0 无法增量备份，5.1 后可以。如果要实现增量备份需要在 `my.ini` 文件中配置备份路径即可，重启 MySQL 服务器，增量备份就启动了。



#### 6. 备份工具的选择？

视库的大小来定，一般来说 100G 内的库，可以考虑使用 mysqldump 来做，因为 mysqldump 更加轻巧灵活，备份时间选在业务低峰期，可以每天进行都进行全量备份(mysqldump 备份出来的文件比较小，压缩之后更小)。

100G 以上的库，可以考虑用 xtrabackup 来做，备份速度明显要比 mysqldump 要快。一般是选择一周一个全备，其余每天进行增量备份，备份时间为业务低峰期。

> 一般情况下，选择每周备份 + 每天增量备份比较靠谱。



#### 7. mysqldump 和 xtrabackup 实现原理？

##### 7.1 mysqldump

mysqldump 是最简单的逻辑备份方式。

- 在备份 MyISAM 表的时候，如果要得到一致的数据，就需要锁表，简单而粗暴。
- 在备份 InnoDB 表的时候，加上 `–master-data=1 –single-transaction` 选项，在事务开始时刻，记录下 binlog pos 点，然后利用 MVCC 来获取一致的数据，由于是一个长事务，在写入和更新量很大的数据库上，将产生非常多的 undo ，显著影响性能，所以要慎用。
- 优点：简单，可针对单表备份，在全量导出表结构的时候尤其有用。
- 缺点：简单粗暴，单线程，备份慢而且恢复慢，跨 IDC 有可能遇到时区问题



##### 7.2 xtrabackup

xtrabackup 实际上是物理备份+逻辑备份的组合。

- 在备份 InnoDB 表的时候，它拷贝 ibd 文件，并一刻不停的监视 redo log 的变化，append 到自己的事务日志文件。在拷贝 ibd 文件过程中，ibd文件本身可能被写”花”，这都不是问题，因为在拷贝完成后的第一个 prepare 阶段，xtrabackup 采用类似于 Innodb 崩溃恢复的方法，把数据文件恢复到与日志文件一致的状态，并把未提交的事务回滚。
- 如果同时需要备份 MyISAM 表以及 InnoDB 表结构等文件，那么就需要用 `flush tables with lock` 来获得全局锁，开始拷贝这些不再变化的文件，同时获得 binlog 位置，拷贝结束后释放锁，也停止对 redo log 的监视。