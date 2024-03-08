搜索的相关性算分，描述了一个文档和查询语句匹配的程度。ES 会对每个匹配查询条件的结果进行算分 _score



#### 1. TF-IDF

打分的本质是排序，需要把最符合用户需求的文档排在前面。ES 5 之前，默认的相关性算分 采用 TF-IDF，现在采用 BM 25



**词频 TF** （Term Frequency）：检索词在一篇文档中出现的频率

检索词出现的次数除以文档的总字数



度量一条查询和结果文档相关性的简单方法：简单将搜索中每一个 词的 TF 进行相加

但是这种情况我们需要考虑到一些Stop Word对最终结果的影响，比如“的”，“的”在文档中出现了很多次，但是对贡献相关度几乎没有用处，不应该考虑他们的 TF。



所以引入了**逆文档频率 ID**F

DF（Document Frequency）是检索词在所有文档中出现的频率，IDF（Inverse Document Frequency）就是逆文档频率

简单说 = log(全部文档数/检索词出现过的文档总数）



TF-IDF 本质上就是将 TF 求和变成了加权求和

比如对“区块链的应用”这句话搜索根据TF-IDF算法就是

![img](http://pcc.huitogo.club/cc56431fdda4202552761471624800fe)



**Lucene 中的 TF-IDF 评分公式**

![img](http://pcc.huitogo.club/00761d2df6d660f47bafbfb63eda1122)



#### 2. BM25

从 ES 5 开始，默认算法改为 BM 25

和经典的TF-IDF相比，当 TF 无限增加时， BM 25算分会趋于一个数值

可以手动配置index的相关性计算方法

![img](http://pcc.huitogo.club/c82371e87bef59c364831defb8cf5947)



BM25的计算公式如下

![img](http://pcc.huitogo.club/13ca0d0c896425492cd8502fbf23d0ba)



其中K 默认值是 1.2，数值越小，饱和度越高，b 默认值是 0.75（取值范围 0-1），0 代表禁止 Normalization