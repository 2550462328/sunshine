集群上的索引会越来越多，例如，你会为你的日志每天创建1个索引

使用多个索引可以让你更好的管理你的数据，提高性能

```
logs-2019-05-01

logs-2019-05-02

logs-2019-05-03
```



因此我们需要设定Index Template，帮助你设定Mappings和Settings，并**按照一定的规则，自动匹配到新创建的索引之上**

1. 模版仅在一个索引被新创建时，才会产生作用。修改模版不会影响已创建的索引
2. 你可以设定多个索引模版，这些设置会被“merge”在一 起
3. 你可以指定“order” 的数值，控制“merge”的过程



例如可以创建两个Index Template匹配到同一个index格式，按照order的顺序进行”merge”

![img](http://pcc.huitogo.club/65a4e5ffc1fcbd4e0bbe2390284fd31e)

![img](http://pcc.huitogo.club/ecb1e98291c5cc242132d21d0f486b0f)



**Index Template的工作方式是什么？**

当一个索引被新创建时

1.  应用Elasticsearch默认的settings和mappings
2.  应用order数值低的Index Template中的设定
3.  应用order高的Index Template中的设定，之前的设定会被覆盖
4.  应用创建索引时，用户所指定的Settings和Mappings, 并覆盖之前模版中的设定



**什么是Dynamic Template？**

根据Elasticsearch识别的数据类型，结合字段名称，来动态设定字段类型，例如：

-  所有的字符串类型都设定成Keyword, 或者关闭keyword字段
-  is开头的字段都设置成boolean
-  long_开头的都设置成long类型



创建Dynamic Template过程格式如下，注意Dynamic Template是定义在在某个索引的Mapping中

![img](http://pcc.huitogo.club/33e97c8d9585bdfe94990b18741a3095)