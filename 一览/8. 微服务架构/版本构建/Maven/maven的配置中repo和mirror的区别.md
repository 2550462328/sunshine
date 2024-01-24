#### 1. Maven仓库分类

![img](http://pcc.huitogo.club/1d2c277c0068f155123eb948bce65c7f)



其中：

- remote repository：相当于公共的仓库，大家都能访问到，一般可以用URL的形式访问
- local repository：存放在本地磁盘的一个文件夹，例如，windows上默认是C:Users｛用户名｝.m2repository目录
- 中央仓库：http://repo1.maven.org/maven2/ 
- 私服：内网自建的maven repository，其URL是一个内部网址 

- 其他公共仓库：其他可以互联网公共访问maven repository，例如 jboss repository等



repository里存放的都是各种jar包和maven插件。当向仓库请求插件或依赖的时候，会先检查local repository，如果local repository有则直接返回，否则会向remote repository请求，并缓存到local repository。也可以把做的东西放到本地仓库，仅供本地使用；或上传到远程仓库，供大家使用。 



#### 2. Mirror

mirror相当于一个拦截器，它会拦截maven对remote repository的相关请求，把请求里的remote repository地址，重定向到mirror里配置的地址。



没有配置mirror：

![img](http://pcc.huitogo.club/8887f9567945c1f3edbd570548f1c238)



配置了mirror后：

![img](http://pcc.huitogo.club/d5d09c83ebd56b6d6dc48634a2b2caaf)



此时，B Repository被称为A Repository的镜像。

如果仓库X可以提供仓库Y存储的所有内容，那么就可以认为X是Y的一个镜像。换句话说，任何一个可以从仓库Y获得的构件，都能够从它的镜像中获取。



在mirror配置最重要的就是mirrorof，就是指定它是哪个仓库的镜像

配置规则如下：

```
<mirrorOf>*</mirrorOf> 
```

匹配所有远程仓库。 



```
<mirrorOf>repo1,repo2</mirrorOf>
```

匹配仓库repo1和repo2，使用逗号分隔多个远程仓库。 



```
<mirrorOf>*,!repo1</miiroOf> 
```

匹配所有远程仓库，repo1除外，使用感叹号将仓库从匹配中排除。



#### 3. repository和mirror

其实，mirror表示的是两个Repository之间的关系，在maven配置文件（setting.xml)里配置了......，即定义了两个Repository之间的镜像关系。

使用mirror的目的大多数是为了出于访问速度和下载速度考虑。

需要注意的是，由于镜像仓库完全屏蔽了被镜像仓库，当镜像仓库不稳定或者停止服务的时候，Maven仍将无法访问被镜像仓库，因而将无法下载构件。



注意：配置Repository 也要配置pluginRepository，否则就直接配置mirrorof 全部映射转发