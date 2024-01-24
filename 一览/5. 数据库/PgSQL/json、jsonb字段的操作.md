#### 1. json 和jsonb 区别

两者从用户操作的角度来说没有区别，区别主要是存储和读取的系统处理（预处理）和耗时方面有区别。json写入快，读取慢，jsonb写入慢，读取快。



#### 2. 常用的操作符

示例说明

```
表名：test_t
json类型的字段名：test_json
字段值：{"a1":"b1","a2":{"c1":"d1","c2":"d2"}}
```



操作符

- ->：用于通过 key 获取json或jsonb对象字段的值，返回值类型为json或jsonb

![img](http://pcc.huitogo.club/945fff70d4789f13976c05b8ff8f82e1)



- ->>：用于通过 key 获取json或jsonb对象字段的值，返回值类型为text

![img](http://pcc.huitogo.club/af6aa69e831d4daccc5b1e09c6b55a7c)



- \#>：用于通过指定提取key的顺序，获取指定的值，返回类型为json或jsonb

![img](http://pcc.huitogo.club/26589b3cdb00bb060edcc71e98b5811f)



- \#>>：用于通过指定提取key的顺序，获取指定的值，返回值类型为text

![img](http://pcc.huitogo.club/57ec2b68936b57edf6ee2a321491a1f2)



- -> 和->> 组合查询复杂的json对象

![img](http://pcc.huitogo.club/47a30c56b07093b2d5b9f07cab19f2b3)



#### 3. PostgreSQL 提供的JSON 处理函数

##### 3.1 json_each

- json_each ( json ) → setof record ( key text , value json )
- jsonb_each ( jsonb ) → setof record ( key text , value jsonb )



功能：key,value以记录的形式返回，value类型为json或jsonb

示例： select * from json_each ((select test_json AS tv from test_t ));

![img](http://pcc.huitogo.club/03c75b0674bc14ebd59c797285be7be9)



##### 3.2  json_each_text

- json_each_text ( json ) → setof record ( key text , value text )
- jsonb_each_text ( jsonb ) → setof record ( key text , value text )



功能：key,value以记录的形式返回，value类型为text

示例：select * from json_each_text ((select test_json AS tv from test_t ));

![img](http://pcc.huitogo.club/1a319c90f93646a3285096b3a3953a84)



##### 3.3 json_extract_path

- json_extract_path ( from_json json , VARIADIC path_elems text[] ) → json
- jsonb_extract_path ( from_json jsonb , VARIADIC path_elems text[] ) → jsonb



功能：按照指定的key顺序，查找值，值的类型为json或jsonb。类似#>

示例：select * from json_extract_path((select test_json AS tv from test_t),'a2', 'c1');

![img](http://pcc.huitogo.club/681add6e86d4fb08bba2100ecd0d50ea)



##### 3.4 json_extract_path_text

- json_extract_path_text ( from_json json , VARIADIC path_elems text[] ) → text
- jsonb_extract_path_text ( from_json jsonb , VARIADIC path_elems text[] ) → text



功能：按照指定的key顺序，查找值，值的类型为text。类似#>>

示例：select * from json_extract_path_text ((select test_json AS tv from test_t),'a2', 'c1');

![img](http://pcc.huitogo.club/bd5076717fe232b3a2ce4c200cceafbc)



##### 3.5 json_object_keys 

- json_object_keys ( json ) → setof text
- jsonb_object_keys ( jsonb ) → setof text



功能：返回json对象的key

示例：select * from json_object_keys ((select test_json AS tv from test_t));

![img](http://pcc.huitogo.club/f9d9aa8c259b56e429dae5dacd8c879b)