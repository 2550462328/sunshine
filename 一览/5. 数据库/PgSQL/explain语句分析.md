expain 语句执行如下

EXPLAIN [ ( option [, ...] ) ] statement



其中option的值有：

-  **ANALYZE** [ boolean ]：通过实际执行的sql来获取相应的计划。这个是真正执行的，多以可以真是的看到执行计划花费了多少的时间，还有它返回的行数
- **VERBOSE** [ boolean ]：用于显示计划的附加信息。这些附加的信息有：计划中每个节点输出的各个列，如果触发器被触发，还会输出触发器的名称。
- **COSTS** [ boolean ]：显示每个计划节点的启动成本和总成本，以及估计行数和每行宽度。该选项默认是TRUE。
- **BUFFERS** [ boolean ]：显示关于缓存区的信息。该选项只能与ANALYZE参数一起使用。显示的缓存区信息包括共享块，本地块和临时读和写的块数。共享块、本地块和临时块分别包含表和索引、临时快和临时索引、以及在排序和物化计划中使用的磁盘块。上层节点显示出来的数据块包含其所有子节点使用的块数。该选项默认为FALSE。
- **TIMING** [ boolean ]：记录下SQL每个NODE的执行时间。
- **FORMAT** { TEXT | XML | JSON | YAML }：表示输出结果格式，默认为TRADITIONAL（表格）



其中我们主要关注的是ANALYZE命令的返回值

先看下正常explain语句的返回项

```
1. explain select * from test1 

2.  

3. QUERY PLAN 

4. ----------------------------------------------------------------------------------------------------------------------- 

5. Seq Scan on test1 (cost=0.00..146666.56 rows=7999956 width=33) 
```

- **Seq Scan**：表示的是全表扫描
- **cost=0.00..146666.56**：cost后面有两个数字,中间使用..分割，第一个数字0.00表示启动的成本，也就是返回第一行需要多少cost值；第二行表示返回所有数据的成本
- **rows=7999956**：表示会返回7999956行
- **width=33**：表示每行的数据宽度为33字节



再看下explain analyze命令的返回项

```
1. explain analyze select * from test1 

2.  

3. QUERY PLAN 

4. ----------------------------------------------------------------------------------------------------------------------- 

5. Seq Scan on test1 (cost=0.00..146666.56 rows=7999956 width=33) (actual time=0.012..1153.317 rows=8000001 loops=1) 

6. Planning time: 0.049 ms 

7. Execution time: 1637.480 ms 
```



可以看到加了analyze后可以看到实际的执行成本

- **actual time=0.012..1153.317:0.01**：表示的是启动的时间,..后面的时间表示返回所有行需要的时间
- **rows=8000001**：表示返回的行数



在介绍explain的原理和优化策略之前我们先了解一些知识点

**执行器扫描策略**：

- **全表扫描**

   全表扫描在pgsql中叫做顺序扫描(seq scan)，全表扫描就是把表的的所有的数据从头到 尾读取一遍，然后从数据块中找到符合条件的数据块。

- **索引扫描**

  索引是为了加快数据查询的速度索引而增加的(Index Scan)。

- **位图扫描**

  位图扫描也是走索引的一种方式。方法是扫描索引，那满足条件的行或者块在内存中建立一个位图，扫描完索引后，再根据位图到表的数据文件中把相应的数据读出来。如果走了两个索引，可以把两个索引进行’and‘或‘or’计算，合并到一个位图，再到表的数据文件中把数据读出来。



**连表查询策略：**

| 类别     | Nested Loop                                                  | Hash Join                                                    | Merge Join                                                   |
| :------- | :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| 使用条件 | 任何条件                                                     | 等值连接（=）                                                | 等值或非等值连接(>，<，=，>=，<=)，‘<>’除外                  |
| 相关资源 | CPU、磁盘I/O                                                 | 内存、临时空间                                               | 内存、临时空间                                               |
| 特点     | 当有高选择性索引或进行限制性搜索时效率比较高，能够快速返回第一次的搜索结果。 | 当缺乏索引或者索引条件模糊时，Hash Join比Nested Loop有效。通常比Merge Join快,如果有索引，或者结果已经被排序了，这时候Merge Join的查询更快。在数据仓库环境下，如果表的纪录数多，效率高。 | 当缺乏索引或者索引条件模糊时，Merge Join比Nested Loop有效。非等值连接时，Merge Join比Hash Join更有效 |
| 缺点     | 当索引丢失或者查询条件限制不够时，效率很低；当表的纪录数多时，效率低。 | 为建立哈希表，需要大量内存。第一次的结果返回较慢。           | 所有的表都需要排序。它为最优化的吞吐量而设计，并且在结果没有全部找到前不返回数据。 |



下面我们开始介绍explain sql语句的原理和可能返回项目



#### 1. Explain基础

查询规划是以规划为节点的树形结构。树的最底节点是扫描节点：他返回表中的原数据行。

不同的表有不同的扫描节点类型：顺序扫描，索引扫描和位图索引扫描。

也有非表列源，如VALUES子句并设置FROM返回，他们有自己的扫描类型。

如果查询需要关联，聚合，排序或其他操作，会在扫描节点之上增加节点执行这些操作。通常有不止一种可能的方式做这些操作，所以可能出现不同的节点类型。

EXPLAIN的输出是每个树节点显示一行，内容是基本节点类型和执行节点的消耗评估。可能会出现其他行，从汇总行节点缩进显示节点的其他属性。第一行（最上节点的汇总行）是评估执行计划的总消耗，这个值越小越好。



#### 2. Where语句

下面是一个简单的例子：

```
1. EXPLAIN SELECT * FROM tenk1;  

2.   

3.             QUERY PLAN  

4. -------------------------------------------------------------  

5. Seq Scan on tenk1 (cost=0.00..458.00 rows=10000 width=244)  
```



因为这个查询没有WHERE子句，所以必须扫描表中的所有行，所以规划器选择使用简单的顺序扫描规划。括号中的数字从左到右依次是：

1.  评估开始消耗。这是可以开始输出前的时间，比如排序节点的排序的时间。
2.  评估总消耗。假设查询从执行到结束的时间。有时父节点可能停止这个过程，比如LIMIT子句。
3.  评估查询节点的输出行数，假设该节点执行结束。
4.  评估查询节点的输出行的平均字节数。



这个消耗的计算依赖于规划器的设置参数，这里的例子都是在默认参数下运行。

需要知道的是：上级节点的消耗包括其子节点的消耗。这个消耗值只反映规划器关心的内容，一般这个消耗不包括将数据传输到客户端的时间。



评估的行数不是执行和扫描查询节点的数量，而是节点返回的数量。它通常会少于扫描数量，因为有WHERE条件会过滤掉一些数据。理想情况顶级行数评估近似于实际返回的数量

回到刚才的例子，表tenk1有10000条数据分布在358个磁盘页，评估时间是（磁盘页*seq_page_cost）+（扫描行*cpu_tuple_cost）。默认seq_page_cost是1.0，cpu_tuple_cost是0.01，所以评估值是(358 * 1.0) + (10000 * 0.01) = 458



现在我们将查询加上WHERE子句：

```
1. EXPLAIN SELECT * FROM tenk1 WHERE unique1 < 7000;  

2.   

3.             QUERY PLAN  

4. ------------------------------------------------------------  

5. Seq Scan on tenk1 (cost=0.00..483.00 rows=7001 width=244)  

6.  Filter: (unique1 < 7000)  
```



查询节点增加了“filter”条件。这意味着查询节点为扫描的每一行数据增加条件检查，只输入符合条件数据。评估的输出记录数因为where子句变少了，但是扫描的数据还是10000条，所以消耗没有减少，反而增加了一点cpu的计算时间。

这个查询实际输出的记录数是7000，但是评估是个近似值，多次运行可能略有差别，这种情况可以通过ANALYZE命令改善。



#### 4. Index

现在再修改一下条件

```
1. EXPLAIN SELECT * FROM tenk1 WHERE unique1 < 100;  

2.   

3.                  QUERY PLAN  

4. ------------------------------------------------------------------------------  

5. Bitmap Heap Scan on tenk1 (cost=5.07..229.20 rows=101 width=244)  

6.  Recheck Cond: (unique1 < 100)  

7.  -> Bitmap Index Scan on tenk1_unique1 (cost=0.00..5.04 rows=101 width=0)  

8.     Index Cond: (unique1 < 100)  
```



查询规划器决定使用两步规划：首先子查询节点根据索引找到符合条件的记录索引，然后外层查询节点将这些记录从表中提取出来。分别提取数据的成本要高于顺序读取，但因为不需要读取所有磁盘页，所以总消耗比较小。（其中Bitmap是系统排序的一种机制）



现在，增加另一个查询条件：

```
1. EXPLAIN SELECT * FROM tenk1 WHERE unique1 < 100 AND stringu1 = 'xxx';  

2.   

3.                  QUERY PLAN  

4. ------------------------------------------------------------------------------  

5. Bitmap Heap Scan on tenk1 (cost=5.04..229.43 rows=1 width=244)  

6.  Recheck Cond: (unique1 < 100)  

7.  Filter: (stringu1 = 'xxx'::name)  

8.  -> Bitmap Index Scan on tenk1_unique1 (cost=0.00..5.04 rows=101 width=0)  

9.     Index Cond: (unique1 < 100)  
```



增加的条件stringu1='xxx'减少了输出记录数的评估，但没有减少时间消耗，因为系统还是要查询相同数量的记录。请注意stringu1不是索引条件。



#### 4. 组合索引

如果在不同的字段上有独立的索引，规划器可能选择使用AND或者OR组合索引：

```
1. EXPLAIN SELECT * FROM tenk1 WHERE unique1 < 100 AND unique2 > 9000;  

2.   

3.                   QUERY PLAN  

4. -------------------------------------------------------------------------------------  

5. Bitmap Heap Scan on tenk1 (cost=25.08..60.21 rows=10 width=244)  

6.  Recheck Cond: ((unique1 < 100) AND (unique2 > 9000))  

7.  -> BitmapAnd (cost=25.08..25.08 rows=10 width=0)  

8.     -> Bitmap Index Scan on tenk1_unique1 (cost=0.00..5.04 rows=101 width=0)  

9.        Index Cond: (unique1 < 100)  

10.     -> Bitmap Index Scan on tenk1_unique2 (cost=0.00..19.78 rows=999 width=0)  

11.        Index Cond: (unique2 > 9000)  
```

这个查询条件的两个字段都有索引，索引不需要filter。



#### 5. Limit

下面我们来看看LIMIT的影响：

```
1. EXPLAIN SELECT * FROM tenk1 WHERE unique1 < 100 AND unique2 > 9000 LIMIT 2;  

2.   

3.                   QUERY PLAN  

4. -------------------------------------------------------------------------------------  

5. Limit (cost=0.29..14.48 rows=2 width=244)  

6.  -> Index Scan using tenk1_unique2 on tenk1 (cost=0.29..71.27 rows=10 width=244)  

7.     Index Cond: (unique2 > 9000)  

8.     Filter: (unique1 < 100)  
```

这条查询的where条件和上面的一样，只是增加了LIMIT，所以不是所有数据都需要返回，规划器改变了规划。索引扫描节点总消耗和返回记录数是运行完查询之后的数值，但Limit节点预期时间消耗是15，所以总时间消耗是15。增加LIMIT会使启动时间小幅增加（0.25->0.29）。



#### 6. Nested Loop

来看一下通过索引字段的表连接：

```
1. EXPLAIN SELECT *  

2. FROM tenk1 t1, tenk2 t2  

3. WHERE t1.unique1 < 10 AND t1.unique2 = t2.unique2;  

4.   

5.                    QUERY PLAN  

6. --------------------------------------------------------------------------------------  

7. Nested Loop (cost=4.65..118.62 rows=10 width=488)  

8.  -> Bitmap Heap Scan on tenk1 t1 (cost=4.36..39.47 rows=10 width=244)  

9.     Recheck Cond: (unique1 < 10)  

10.     -> Bitmap Index Scan on tenk1_unique1 (cost=0.00..4.36 rows=10 width=0)  

11.        Index Cond: (unique1 < 10)  

12.  -> Index Scan using tenk2_unique2 on tenk2 t2 (cost=0.29..7.91 rows=1 width=244)  

13.     Index Cond: (unique2 = t1.unique2) 
```

这个规划中有一个内连接的节点，它有两个子节点。节点摘要行的缩进反映了规划树的结构。最外层是一个连接节点，子节点是一个Bitmap扫描。外部节点位图扫描的消耗和记录数如同我们使用SELECT...WHERE unique1 < 10，因为这时t1.unique2 = t2.unique2还不相关。接下来为每一个从外部节点得到的记录运行内部查询节点。这里外部节点得到的数据的t1.unique2值是可用的，所以我们得到的计划和SELECT...WHEREt2.unique2=constant的情况类似。（考虑到缓存的因素评估的消耗可能要小一些）



外部节点的消耗加上循环内部节点的消耗（39.47+10*7.91）再加一点CPU时间就得到规划的总消耗。



再看一个例子：

```
1. EXPLAIN SELECT *  

2. FROM tenk1 t1, tenk2 t2  

3. WHERE t1.unique1 < 10 AND t2.unique2 < 10 AND t1.hundred < t2.hundred;  

4.   

5.                     QUERY PLAN  

6. ---------------------------------------------------------------------------------------------  

7. Nested Loop (cost=4.65..49.46 rows=33 width=488)  

8.  Join Filter: (t1.hundred < t2.hundred)  

9.  -> Bitmap Heap Scan on tenk1 t1 (cost=4.36..39.47 rows=10 width=244)  

10.     Recheck Cond: (unique1 < 10)  

11.     -> Bitmap Index Scan on tenk1_unique1 (cost=0.00..4.36 rows=10 width=0)  

12.        Index Cond: (unique1 < 10)  

13.  -> Materialize (cost=0.29..8.51 rows=10 width=244)  

14.     -> Index Scan using tenk2_unique2 on tenk2 t2 (cost=0.29..8.46 rows=10 width=244)  

15.        Index Cond: (unique2 < 10)  
```



条件t1.hundred<t2.hundred不在tenk2_unique2索引中，所以这个条件出现在连接节点中。这将减少连接节点的评估输出记录数，但不会改变子节点的扫描数。



注意这次规划器选择使用Meaterialize节点，将条件加入内部节点，这意味着内部节点的索引扫描只做一次，即使嵌套循环需要读取这些数据10次，Meterialize节点将数据保存在内存中，每次循环都从内存中读取数据。



#### 7. Hash Join

如果我们稍微改变一下查询，会看到完全不同的规划：

```
1. EXPLAIN SELECT *  

2. FROM tenk1 t1, tenk2 t2  

3. WHERE t1.unique1 < 100 AND t1.unique2 = t2.unique2;  

4.   

5.                     QUERY PLAN  

6. ------------------------------------------------------------------------------------------  

7. Hash Join (cost=230.47..713.98 rows=101 width=488)  

8.  Hash Cond: (t2.unique2 = t1.unique2)  

9.  -> Seq Scan on tenk2 t2 (cost=0.00..445.00 rows=10000 width=244)  

10.  -> Hash (cost=229.20..229.20 rows=101 width=244)  

11.     -> Bitmap Heap Scan on tenk1 t1 (cost=5.07..229.20 rows=101 width=244)  

12.        Recheck Cond: (unique1 < 100)  

13.        -> Bitmap Index Scan on tenk1_unique1 (cost=0.00..5.04 rows=101 width=0)  

14.           Index Cond: (unique1 < 100) 
```

这里规划器选择使用hash join，将一个表的数据存入内存中的哈希表，然后扫描另一个表并和哈希表中的每一条数据进行匹配。

注意缩进反应的规划结构。将tenk1表上的bitmap扫描结果作为Hash节点的输入建立哈希表。然后Hash Join节点读取外层子节点的数据，再循环检索哈希表的数据。



#### 8. Merge join

另一个可能的连接类型是merge join：

```
1. EXPLAIN SELECT *  

2. FROM tenk1 t1, onek t2  

3. WHERE t1.unique1 < 100 AND t1.unique2 = t2.unique2;  

4.   

5.                     QUERY PLAN  

6. ------------------------------------------------------------------------------------------  

7. Merge Join (cost=198.11..268.19 rows=10 width=488)  

8.  Merge Cond: (t1.unique2 = t2.unique2)  

9.  -> Index Scan using tenk1_unique2 on tenk1 t1 (cost=0.29..656.28 rows=101 width=244)  

10.     Filter: (unique1 < 100)  

11.  -> Sort (cost=197.83..200.33 rows=1000 width=244)  

12.     Sort Key: t2.unique2  

13.     -> Seq Scan on onek t2 (cost=0.00..148.00 rows=1000 width=244)  
```



Merge Join需要已经排序的输入数据。在这个规划中按正确顺序索引扫描tenk1的数据，但是对onek表执行排序和顺序扫描，因为需要在这个表中查询多条数据。因为索引扫描需要访问不连续的磁盘，所以索引扫描多条数据时会频繁使用排序顺序扫描（Sequential-scan-and-sort）。



有一种方法可以看到不同的规划，就是强制规划器忽略任何策略。例如，如果我们不相信排序顺序扫描（sequential-scan-and-sort）是最好的办法，我们可以尝试这样的做法：

```
1. SET enable_sort = off;  

2.   

3. EXPLAIN SELECT *  

4. FROM tenk1 t1, onek t2  

5. WHERE t1.unique1 < 100 AND t1.unique2 = t2.unique2;  

6.   

7.                     QUERY PLAN  

8. ------------------------------------------------------------------------------------------  

9. Merge Join (cost=0.56..292.65 rows=10 width=488)  

10.  Merge Cond: (t1.unique2 = t2.unique2)  

11.  -> Index Scan using tenk1_unique2 on tenk1 t1 (cost=0.29..656.28 rows=101 width=244)  

12.     Filter: (unique1 < 100)  

13.  -> Index Scan using onek_unique2 on onek t2 (cost=0.28..224.79 rows=1000 width=244)  
```

显示结果表明，规划器认为索引扫描比排序顺序扫描消耗高12%。当然下一个问题就是规划器的评估为什么是正确的。我们可以通过EXPLAIN ANALYZE进行考察。



#### 9、Explain analyze

通过EXPLAIN ANALYZE可以检查规划器评估的准确性。使用ANALYZE选项，EXPLAIN实际运行查询，显示真实的返回记录数和运行每个规划节点的时间，例如我们可以得到下面的结果：

```
1. EXPLAIN ANALYZE SELECT *  

2. FROM tenk1 t1, tenk2 t2  

3. WHERE t1.unique1 < 10 AND t1.unique2 = t2.unique2;  

4.   

5.                              QUERY PLAN  

6. ---------------------------------------------------------------------------------------------------------------------------------  

7. Nested Loop (cost=4.65..118.62 rows=10 width=488) (actual time=0.128..0.377 rows=10 loops=1)  

8.  -> Bitmap Heap Scan on tenk1 t1 (cost=4.36..39.47 rows=10 width=244) (actual time=0.057..0.121 rows=10 loops=1)  

9.     Recheck Cond: (unique1 < 10)  

10.     -> Bitmap Index Scan on tenk1_unique1 (cost=0.00..4.36 rows=10 width=0) (actual time=0.024..0.024 rows=10 loops=1)  

11.        Index Cond: (unique1 < 10)  

12.  -> Index Scan using tenk2_unique2 on tenk2 t2 (cost=0.29..7.91 rows=1 width=244) (actual time=0.021..0.022 rows=1 loops=10)  

13.     Index Cond: (unique2 = t1.unique2)  

14. Total runtime: 0.501 ms  
```

注意，实际时间(actual time)的值是以毫秒为单位的实际时间，cost是评估的消耗，是个虚拟单位时间，所以他们看起来不匹配。

通常最重要的是看评估的记录数是否和实际得到的记录数接近。在这个例子里评估数完全和实际一样，但这种情况很少出现。



某些查询规划可能执行多次子规划。比如之前提过的内循环规划（nested-loop），内部索引扫描的次数是外部数据的数量。在这种情况下，报告显示循环执行的总次数、平均实际执行时间和数据条数。这样做是为了和评估值表示方式一致。由循环次数和平均值相乘得到总消耗时间。



#### 10. Sort和Hash

某些情况EXPLAIN ANALYZE会显示额外的信息，比如sort和hash节点的时候：

```
1. EXPLAIN ANALYZE SELECT *  

2. FROM tenk1 t1, tenk2 t2  

3. WHERE t1.unique1 < 100 AND t1.unique2 = t2.unique2 ORDER BY t1.fivethous;  

4.   

5.                                 QUERY PLAN  

6. --------------------------------------------------------------------------------------------------------------------------------------------  

7. Sort (cost=717.34..717.59 rows=101 width=488) (actual time=7.761..7.774 rows=100 loops=1)  

8.  Sort Key: t1.fivethous  

9.  Sort Method: quicksort Memory: 77kB  

10.  -> Hash Join (cost=230.47..713.98 rows=101 width=488) (actual time=0.711..7.427 rows=100 loops=1)  

11.     Hash Cond: (t2.unique2 = t1.unique2)  

12.     -> Seq Scan on tenk2 t2 (cost=0.00..445.00 rows=10000 width=244) (actual time=0.007..2.583 rows=10000 loops=1)  

13.     -> Hash (cost=229.20..229.20 rows=101 width=244) (actual time=0.659..0.659 rows=100 loops=1)  

14.        Buckets: 1024 Batches: 1 Memory Usage: 28kB  

15.        -> Bitmap Heap Scan on tenk1 t1 (cost=5.07..229.20 rows=101 width=244) (actual time=0.080..0.526 rows=100 loops=1)  

16.           Recheck Cond: (unique1 < 100)  

17.           -> Bitmap Index Scan on tenk1_unique1 (cost=0.00..5.04 rows=101 width=0) (actual time=0.049..0.049 rows=100 loops=1)  

18.              Index Cond: (unique1 < 100)  

19. Total runtime: 8.008 ms 
```



排序节点（Sort）显示排序类型（一般是在内存还是在磁盘）和使用多少内存。哈希节点（Hash）显示哈希桶和批数以及使用内存的峰值。



#### 11. Filter

另一种额外信息是过滤条件过滤掉的记录数：

```
1. EXPLAIN ANALYZE SELECT * FROM tenk1 WHERE ten < 7;  

2.   

3.                        QUERY PLAN  

4. ---------------------------------------------------------------------------------------------------------  

5. Seq Scan on tenk1 (cost=0.00..483.00 rows=7000 width=244) (actual time=0.016..5.107 rows=7000 loops=1)  

6.  Filter: (ten < 7)  

7.  Rows Removed by Filter: 3000  

8. Total runtime: 5.905 ms  
```

这个值在join节点上尤其有价值。"Rows Removed"只有在过滤条件过滤掉数据时才显示。



#### 12. Explain buffers

EXPLAIN还有BUFFERS选项可以和ANALYZE一起使用，来得到更多的运行时间分析：

```
1. EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM tenk1 WHERE unique1 < 100 AND unique2 > 9000;  

2.   

3.                              QUERY PLAN  

4. ---------------------------------------------------------------------------------------------------------------------------------  

5. Bitmap Heap Scan on tenk1 (cost=25.08..60.21 rows=10 width=244) (actual time=0.323..0.342 rows=10 loops=1)  

6.  Recheck Cond: ((unique1 < 100) AND (unique2 > 9000))  

7.  Buffers: shared hit=15  

8.  -> BitmapAnd (cost=25.08..25.08 rows=10 width=0) (actual time=0.309..0.309 rows=0 loops=1)  

9.     Buffers: shared hit=7  

10.     -> Bitmap Index Scan on tenk1_unique1 (cost=0.00..5.04 rows=101 width=0) (actual time=0.043..0.043 rows=100 loops=1)  

11.        Index Cond: (unique1 < 100)  

12.        Buffers: shared hit=2  

13.     -> Bitmap Index Scan on tenk1_unique2 (cost=0.00..19.78 rows=999 width=0) (actual time=0.227..0.227 rows=999 loops=1)  

14.        Index Cond: (unique2 > 9000)  

15.        Buffers: shared hit=5  

16. Total runtime: 0.423 
```

Buffers提供的数据可以帮助确定哪些查询是I/O密集型的。



#### 13、Insert/Update/Delete

请注意EXPLAIN ANALYZE实际运行查询，任何实际影响都会发生。如果要分析一个修改数据的查询又不想改变你的表，你可以使用roll back命令进行回滚，比如：

```
1. BEGIN;  

2.   

3. EXPLAIN ANALYZE UPDATE tenk1 SET hundred = hundred + 1 WHERE unique1 < 100;  

4.   

5.                              QUERY PLAN  

6. --------------------------------------------------------------------------------------------------------------------------------  

7. Update on tenk1 (cost=5.07..229.46 rows=101 width=250) (actual time=14.628..14.628 rows=0 loops=1)  

8.  -> Bitmap Heap Scan on tenk1 (cost=5.07..229.46 rows=101 width=250) (actual time=0.101..0.439 rows=100 loops=1)  

9.     Recheck Cond: (unique1 < 100)  

10.     -> Bitmap Index Scan on tenk1_unique1 (cost=0.00..5.04 rows=101 width=0) (actual time=0.043..0.043 rows=100 loops=1)  

11.        Index Cond: (unique1 < 100)  

12. Total runtime: 14.727 ms  

13.   

14. ROLLBACK;
```

当查询是INSERT,UPDATE或DELETE命令时，在顶级节点是实施对表的变更。在这下面的节点实行定位旧数据计算新数据的工作。所以我们看到一样的bitmap索引扫描，并返回给Update节点。值得注意的是虽然修改数据的节点可能需要相当长的运行时间（在这里它消耗了大部分的时间），规划器却没有在评估时间中添加任何消耗，这是因为更新工作对于任何查询规划都是一样的，所以并不影响规划器的决策。



EXPLAIN ANALYZE的"Total runtime"包括执行启动和关闭时间，以及运行被激发的任何触发器的时间，但不包括分析、重写或规划时间。执行时间包括BEFORE触发器，但不包括AFTER触发器，因为AFTER是在查询运行结束之后才触发的。每个触发器（无论BEFORE还是AFTER）的时间也会单独显示出来。注意，延迟的触发器在事务结束前都不会被执行，所以EXPLAIN ANALYZE不会显示。