#### 1. 什么是倒排索引？

对于一个key-value的map，正排索引是根据key查找value，而倒排索引是根据value来查找key。



#### 2. Elasticsearch存储基本概念

- **Term（单词）**

elasticsearch存储文本的时候会经过Lucene分词器进行分词，一条文本会被拆成多个单词，这一个单词就是一个Term



- **Term Dictionary（单词字典）**

存储Term的集合



- **Term Index（单词索引）**

为了更快的找到某个单词，我们为单词建立索引



- **Posting List（倒排列表）**

倒排列表记录了出现过某个单词的所有文档的文档列表及单词在该文档中出现的位置信息，每条记录称为一个倒排项(Posting)。根据倒排列表，即可获知哪些文档包含某个单词。

实际的倒排列表中并不只是存了文档ID这么简单，还有一些其它的信息，比如：词频（Term出现的次数）、偏移量（offset）等。



如果类比现代汉语词典的话，那么Term就相当于词语，Term Dictionary相当于汉语词典本身，Term Index就相当于词典的目录索引。



#### 3. Term Index的数据结构

为Terms建立索引，最好的就是**B-Tree索引**



我们查找Term的过程跟在MyISAM中记录ID的过程大致是一样的

- MyISAM中，索引和数据是分开，通过索引可以找到记录的地址，进而可以找到这条记录
- 在倒排索引中，通过Term索引可以找到Term在Term Dictionary中的位置，进而找到Posting List，有了倒排列表就可以根据ID找到对应的记录



#### 4. Elasticsearch为什么比MySQL快？

**1）首先看一下Mysql在基于InnoDB和MyISAM下的索引查找过程**

![img](http://pcc.huitogo.club/a5077f4b65fd56757ffeab78f5cea295)



- 对于聚簇索引，如果对ID进行查询，会直接对主键索引进行查找，在叶子节点存放的是该行的数据；如果对Name进行查询，会先在辅助键索引表根据Name进行查找，拿到相应的主键再到主键索引进行查找。
- 对于非聚簇索引，通过主键索引和辅助键索引发生的查询操作是一样的，但是在叶子节点存放的是该行数据在内存中的地址。



**2）再来看一下elasticsearch模糊查询指定文档数据的过程**

![img](http://pcc.huitogo.club/8aa023874f3858cf609440423d13808a)



1. ElasticSearch为每个Field生成倒排索引（Term Index，也是采用b-tree存储结构放在内存中）
2. 为了防止Term Index过大，Term Index是FST形式保存 + 公共前缀压缩 ，极大节省了内存空间
3. 通过Term Index的前缀查找Term Dictionary采用二分法，将时间复杂度减少到O(logn)
4. 通过Term Index查找到Term在Term Dictionary的block位置再去磁盘中去取，极大减小了IO消耗
5. 通过Term 拿到Posting List中关联的文档Id查找文档数据采用的是正排索引。



**3）elasticsearch和mysql对比**

elasticsearch比mysql快是分场景的，比如模糊查询



对于mysql来说肯定是不可能为text类型的字段建立索引，那么对于它来说就是规规矩矩的全文检索进行匹配。

对于elasticsearch来说在你存入elasticsearch数据的时候就已经进行了分词，并且为每个单词都建立了倒排索引。

所以在这种场景下mysql是远不如elasticsearch的。



而对于精确查询，在mysql可以建立索引的情况下，elasticsearch反倒不一定有mysql快。



#### 5. Elasticsearch快速查询原理探究

##### 5.1 什么是FST？

之前提到为了减小Term Index所占的内存大小，我们对Term Index使用了FST形式？



什么是FST？

> FSTs are finite-state machines that map a term (byte sequence) to an arbitrary output



假设我们现在要将mop, moth, pop, star, stop and top(term index里的term前缀)映射到序号：0，1，2，3，4，5(term dictionary的block位置)。最简单的做法就是定义个Map<string, integer="">，大家找到自己的位置对应入座就好了，但从内存占用少的角度想想，有没有更优的办法呢？



答案就是：FST

![img](http://pcc.huitogo.club/21e02b941d49f4b5728f04dfb8bbc62c)

⭕️表示一种状态

–>表示状态的变化过程，上面的字母/数字表示状态变化和权重



将单词分成单个字母通过⭕️和–>表示出来，0权重不显示。如果⭕️后面出现分支，就标记权重，最后整条路径上的权重加起来就是这个单词对应的序号。

FST以字节的方式存储所有的term，这种压缩方式可以有效的缩减存储空间，使得term index足以放进内存，但这种方式也会导致查找时需要更多的CPU资源。



##### 5.2 对Posting List进行压缩

**首先我们为什么需要对Posting List进行压缩？**



我们思考一下如果Elasticsearch需要对同学的性别进行索引（用传统数据库无异于全文检索)，会怎样？如果有上千万个同学，而世界上只有男/女这样两个性别，每个posting list都会有至少百万个文档id。 Elasticsearch是如何有效的对这些文档id压缩的呢？



Frame Of Reference

**增量编码压缩，将大数变小数，按字节存储**



首先，Elasticsearch要求posting list是有序的(为了提高搜索的性能，再任性的要求也得满足)，这样做的一个好处是方便压缩，看下面这个图例：

![img](http://pcc.huitogo.club/fcd68332a1c11bc70eaa98a7d3771896)



原理就是通过增量，将原来的大数变成小数仅存储增量值，再精打细算按bit排好队，最后通过字节存储，而不是大大咧咧的尽管是2也是用int(4个字节)来存储。



##### 5.3 Roaring bitmaps

那对posting list存储的数据结构是什么呢？

**答案就是：Roaring bitmaps**



说到Roaring bitmaps，就必须先从bitmap说起。Bitmap是一种数据结构，假设有某个posting list：

[1,3,4,7,10]



对应的bitmap就是：

[1,0,1,1,0,0,1,0,0,1]



非常直观，用0/1表示某个值是否存在，比如10这个值就对应第10位，对应的bit值是1，这样用一个字节就可以代表8个文档id，旧版本(5.0之前)的Lucene就是用这样的方式来压缩的，但这样的压缩方式仍然不够高效，如果有1亿个文档，那么需要12.5MB的存储空间，这仅仅是对应一个索引字段(我们往往会有很多个索引字段)。于是有人想出了Roaring bitmaps这样更高效的数据结构。



Bitmap的缺点是存储空间随着文档个数线性增长，Roaring bitmaps需要打破这个魔咒就一定要用到某些指数特性：

将posting list按照65535为界限分块，比如第一块所包含的文档id范围在0~65535之间，第二块的id范围是65536~131071，以此类推。再用<商，余数>的组合表示每一组id，这样每组里的id范围都在0~65535内了，剩下的就好办了，既然每组id不会变得无限大，那么我们就可以通过最有效的方式对这里的id存储。

![img](http://pcc.huitogo.club/9dec3c7afe46fc44ed3ab8e1780e7e17)



**为什么是以65535为界限?**

程序员的世界里除了1024外，65535也是一个经典值，因为它=2^16-1，正好是用2个字节能表示的最大数，一个short的存储单位，注意到上图里的最后一行“If a block has more than 4096 values, encode as a bit set, and otherwise as a simple array using 2 bytes per value”，如果是大块，用节省点就用bitset存，小块就豪爽点，2个字节我也不计较了，用一个short[]存着方便。



**那为什么用4096来区分大块还是小块呢？**

个人理解：都说程序员的世界是二进制的，4096*2bytes ＝ 8192bytes < 1KB, 磁盘一次寻道可以顺序把一个小块的内容都读出来，再大一位就超过1KB了，需要两次读。



##### 5.4 联合查询

之前一直讨论单field索引，如果**多个field索引的联合查询，倒排索引如何满足快速查询的要求呢？**

方案就是**利用跳表(Skip list)的数据结构快速做“与”运算**，或者**利用上面提到的bitset按位“与”**



假设有下面三个posting list需要联合索引：

![img](http://pcc.huitogo.club/d59c36bc46500b7d647fee5ab4e781db)



如果使用跳表，对最短的posting list中的每个id，逐个在另外两个posting list中查找看是否存在，最后得到交集的结果。

如果使用bitset，就很直观了，直接按位与，得到的结果就是最后的交集。



#### 6. 总结

Elasticsearch的索引思路:

将**磁盘里的东西尽量搬进内存**，减少磁盘随机读取次数(同时也利用磁盘顺序读特性)，结合各种**压缩算法**减小对内存的占用。。



所以，对于使用Elasticsearch进行索引时需要注意:

- **不需要索引的字段，一定要明确定义**出来，因为默认是自动建索引的
- 同样的道理，对于String类型的字段，**不需要analysis的也需要明确定义**出来，因为默认也是会analysis的
- **选择有规律的ID**很重要，随机性太大的ID(比如java的UUID)不利于查询



关于最后一点，个人认为有多个因素:

其中一个(也许不是最重要的)因素: 上面看到的压缩算法，都是对Posting list里的大量ID进行压缩的，那如果ID是顺序的，或者是有公共前缀等具有一定规律性的ID，压缩比会比较高；

另外一个因素: 可能是最影响查询性能的，应该是最后通过Posting list里的ID到磁盘中查找Document信息的那步，因为Elasticsearch是分Segment存储的，根据ID这个大范围的Term定位到Segment的效率直接影响了最后查询的性能，如果ID是有规律的，可以快速跳过不包含该ID的Segment，从而减少不必要的磁盘读次数