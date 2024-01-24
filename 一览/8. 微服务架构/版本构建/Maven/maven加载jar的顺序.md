1. 本地仓库
2. maven settings profile中的repository；
3. pom.xml中profile中定义的repository；
4. pom.xml中的repositorys(定义多个repository，按定义顺序找)；
5. mirror



但是如果mirror中定义了

```
1.  <mirrors>   

2.    <mirror>   

3.      <id>my_mirror</id>    

4.      <name>my_mirror</name>    

5.      <url>http://nexus.xx.yy/nexus/content/groups/public/</url>    

6.      <mirrorOf>*</mirrorOf>   

7.    </mirror>   

8.  </mirrors>  
```

那么其他仓库配的地址，都会失效了，以这个为准。