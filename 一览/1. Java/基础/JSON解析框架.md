#### 1. 什么是BSON

在说json解析方式之前，介绍一下从JSON衍生出的BSON

参考百科说明：BSON( Binary Serialized Document Format) 是一种二进制形式的存储格式，采用了类似于 C 语言结构体的名称、对表示方法，支持内嵌的文档对象和数组对象，具有轻量性、可遍历性、高效性的特点，可以有效描述非结构化数据和结构化数据。

BSON是一种类json的一种二进制形式的存储格式，简称Binary JSON，它和JSON一样，支持内嵌的文档对象和数组对象，但是BSON有JSON没有的一些数据类型，如Date和BinData类型。BSON可以做为网络数据交换的一种存储形式，这个有点类似于Google的Protocol Buffer，但是BSON是一种schema-less的存储形式，它的优点是灵活性高，但它的缺点是空间利用率不是很理想。

BSON有三个特点：**轻量性**、**可遍历性**、**高效性**。

例子A 简单的Document：

```
1.  {  

2.       title:"MongoDB",  

3.      last_editor:"192.168.1.122",  

4.      last_modified:new Date("27/06/2011"),  

5.      body:"MongoDB introduction",  

6.      categories:["Database","NoSQL","BSON"],  

7.      revieved:false  

8.  }  
```



例子B 嵌套的例子：

```
1.  {  

2.      name:"lemo",  

3.      age:"12",  

4.      address:{  

5.          city:"suzhou",  

6.          country:"china",  

7.          code:215000  

8.      } ,  

9.      scores:[  

10.         {"name":"english","grade:3.0},  

11.         {"name":"chinese","grade:2.0}  

12.     ]  

13. }  
```

这种嵌套结构用关系数据库做还是比较复杂的。



##### 1.1 BSON和JSON的区别

BSON是由10gen开发的一个数据格式，目前主要用于MongoDB中，是mongodb的数据存储格式。BSON基于JSON格式，选择JSON进行改造的原因主要是JSON的通用性及JSON的schemaless的特性。



BSON主要会实现以下三点目标：

1. **更快的遍历速度**

   对JSON格式来说，太大的JSON结构会导致数据遍历非常慢。在JSON中，要跳过一个文档进行数据读取，需要对此文档进行扫描才行，需要进行麻烦的数据结构匹配，比如括号的匹配，而BSON对JSON的一大改进就是，它会将JSON的每一个元素的长度存在元素的头部，这样你只需要读取到元素长度就能直接seek到指定的点上进行读取了。

2. **操作更简易**

   对JSON来说，数据存储是无类型的，比如你要修改基本一个值，从9到10，由于从一个字符变成了两个，所以可能其后面的所有内容都需要往后移一位才可以。而使用BSON，你可以指定这个列为数字列，那么无论数字从9长到10还是100，我们都只是在存储数字的那一位上进行修改，不会导致数据总长变大。当然，在MongoDB中，如果数字从整形增大到长整型，还是会导致数据总长变大的。

3. **增加了额外的数据类型**

   JSON是一个很方便的数据交换格式，但是其类型比较有限。BSON在其基础上增加了“byte array”数据类型。这使得二进制的存储不再需要先base64转换后再存成JSON。大大减少了计算开销和数据大小。

但是，在有的时候，BSON相对JSON来说也并没有空间上的优势，比如对{“field”:7}，在JSON的存储上7只使用了一个字节，而如果用BSON，那就是至少4个字节（32位）

目前在10gen的努力下，BSON已经有了针对多种语言的编码解码包。并且都是Apache 2 license下开源的。并且还在随着MongoDB进一步地发展。



认识完BSON之后下面比较JSON常见的几种解析方式

#### 2. json-lib

json-lib最开始的也是应用最广泛的json解析工具，json-lib 不好的地方确实是依赖于很多第三方包，

包括commons-beanutils.jar，commons-collections-3.2.jar，commons-lang-2.6.jar，commons-logging-1.1.1.jar，ezmorph-1.0.6.jar，

对于复杂类型的转换，json-lib对于json转换成bean还有缺陷，比如一个类里面会出现另一个类的list或者map集合，json-lib从json到bean的转换就会出现问题。

json-lib在功能和性能上面都不能满足现在互联网化的需求。



#### 3. 开源的Jackson

相比json-lib框架，Jackson所依赖的jar包较少，简单易用并且性能也要相对高些。

而且Jackson社区相对比较活跃，更新速度也比较快。

Jackson对于复杂类型的json转换bean会出现问题，一些集合Map，List的转换出现问题。

Jackson对于复杂类型的bean转换Json，转换的json格式不是标准的Json格式



#### 4. Google的Gson

Gson是目前功能最全的Json解析神器，Gson当初是为因应Google公司内部需求而由Google自行研发而来，

但自从在2008年五月公开发布第一版后已被许多公司或用户应用。

Gson的应用主要为toJson与fromJson两个转换函数，无依赖，不需要例外额外的jar，能够直接跑在JDK上。

而在使用这种对象转换之前需先创建好对象的类型以及其成员才能成功的将JSON字符串成功转换成相对应的对象。

类里面只要有get和set方法，Gson完全可以将复杂类型的json到bean或bean到json的转换，是JSON解析的神器。

Gson在功能上面无可挑剔，但是性能上面比FastJson有所差距。



#### 5. 阿里巴巴的FastJson

Fastjson是一个Java语言编写的高性能的JSON处理器,由阿里巴巴公司开发。

无依赖，不需要例外额外的jar，能够直接跑在JDK上。

FastJson在复杂类型的Bean转换Json上会出现一些问题，可能会出现引用的类型，导致Json转换出错，需要制定引用。

FastJson采用独创的算法，将parse的速度提升到极致，超过所有json库。



综上4种Json技术的比较，在项目选型的时候可以使用Google的Gson和阿里巴巴的FastJson两种并行使用。

- 如果只是功能要求，没有性能要求，可以使用google的Gson。
- **如果有性能上面的要求可以使用Gson将bean转换json确保数据的正确，使用FastJson将Json转换Bean。**