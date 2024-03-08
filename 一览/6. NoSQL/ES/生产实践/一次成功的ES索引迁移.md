记录一次成功的elasticsearch数据迁移，将windows上的elasticsearch迁移到linux上，两个elasticsearch的版本都是7.5.0。

使用的方式是基于快照的方式，本地生成一个数据快照，然后发送到远程，然后远程再存储一下这个快照就可以了

#### 1. 在windows端

1）声明本地快照仓库地址

需要修改config下的elasticsearch.yml文件，修改如下

![img](http://pcc.huitogo.club/cd1d04bc756f62fd23328f0703e541c2)

这里/backup的目录在elasticsearch安装目录下的根目录下，比如elasticsearch安装在C盘下，这里的/backup就是C:/backup；在linux下不会这样



2）注册快照仓库

可以通过curl的方式，我这里使用的是postman

![img](http://pcc.huitogo.club/df11aae2d98fac39986647ab15509763)

这里my_backup就是仓库名称



可以通过下面验证

![img](http://pcc.huitogo.club/76b2597b6e8bb0a2a66000e748ea0add)



3）生成快照

![img](http://pcc.huitogo.club/0153c70e698d13a14dfba9fc3be90bc7)

这里index_name就是索引名称了，如果不知道索引名称的话可以搭建Kibana查看。

然后进入/backup所在的目录，即可看到生成的索引文件。



#### 2. linux端

1）声明本地快照仓库地址

修改elasticsearch.yml文件

![img](http://pcc.huitogo.club/1cc3ccf5c0727ec4f9a3f8c60006f5d6)



2）注册快照仓库

```
curl -XPUT http://127.0.0.1:9200/_snapshot/my_backup -H "Content-Type: application/json" -d '{"type": "fs","settings":{"location":"/usr/backup","compress":"true"}}'
```

这里使用curl方式，my_backup仓库名称，不要求和windows端一致，主要是location。

然后将windows生成的索引文件通过ftp工具传到Linux端仓库中，这里是/usr/backup下。



3）存储快照

```
curl -XPOST http://127.0.0.1:9200/_snapshot/my_backup/index_name/_restore
```

这里index_name就是索引名称了。



如果显示index_name存在的情况下，需要先删除索引

```
curl -XDELETE http://127.0.0.1:9200/post
```



到此完成了。