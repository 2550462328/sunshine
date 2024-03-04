#### 1. 什么是shell

![img](http://pcc.huitogo.club/7fb50f3b8f1bb233d1759f3cac5197d1)



#### 2. Shell的helloworld

1）编辑helloworld.sh如下

```
1. #!/bin/bash 

2. #第一个shell小程序,echo 是linux中的输出命令。 

3. echo "helloworld!"
```



shell中 # 符号表示注释。shell 的第一行比较特殊，一般都会以#!开始来指定使用的 shell 类型。在linux中，除了bash shell以外，还有很多版本的shell， 例如zsh、dash等等...不过bash shell还是我们使用最多的。



2）运行脚本:./helloworld.sh 。（注意，一定要写成 ./helloworld.sh ，而不是 helloworld.sh ，运行其它二进制的程序也一样，直接写 helloworld.sh ，linux 系统会去 PATH 里寻找有没有叫 helloworld.sh 的，而只有 /bin, /sbin, /usr/bin，/usr/sbin 等在 PATH 里，你的当前目录通常不在 PATH 里，所以写成 helloworld.sh 是会找不到命令的，要用./helloworld.sh 告诉系统说，就在当前目录找。）



#### 3. shell变量

Shell编程中一般分为三种变量：

1. 我们自己定义的变量（自定义变量）: 仅在当前 Shell 实例中有效，其他 Shell 启动的程序不能访问局部变量。
2. Linux已定义的环境变量（环境变量， 例如：PATH,PATH,HOME 等..., 这类变量我们可以直接使用），使用 env 命令可以查看所有的环境变量，而set命令既可以查看环境变量也可以查看自定义变量。
3. Shell变量 ：Shell变量是由 Shell 程序设置的特殊变量。Shell 变量中有一部分是环境变量，有一部分是局部变量，这些变量保证了 Shell 的正常运行

```
1. #!/bin/bash 

2. #自定义变量hello 

3. hello="hello world" 

4. echo $hello 

5.  

6. #linux环境变量 

7. echo $HOME 
```



Shell 编程中的变量名的命名的注意事项：

- 命名只能使用英文字母，数字和下划线，首个字符不能以数字开头，但是可以使用下划 线（_）开头。
- 中间不能有空格，可以使用下划线（_）。
- 不能使用标点符号。
- 不能使用bash里的关键字（可用help命令查看保留关键字）。



#### 4. shell字符串操作

- **拼接字符串**

```
1. #!/bin/bash 

2. name="SnailClimb" 

3. # 使用双引号拼接 

4. greeting="hello, "$name" !" 

5. greeting_1="hello, ${name} !" 

6. echo $greeting $greeting_1 

7. # 使用单引号拼接 

8. greeting_2='hello, '$name' !' 

9. greeting_3='hello, ${name} !' 

10. echo $greeting_2 $greeting_3 
```



- **获取字符串长度**

```
1. #!/bin/bash 

2. #获取字符串长度 

3. name="SnailClimb" 

4. # 第一种方式 

5. echo ${#name} #输出 10 

6. # 第二种方式 

7. expr length "$name"; 
```



- **计算字符串（expr命令）**

```
1. #表达式中的运算符左右必须包含空格，如果不包含空格，将会输出表达式本身 

2. expr 5+6  // 直接输出 5+6 

3. expr 5 + 6    // 输出 11 

4.  

5. #某些运算符，还需要我们使用符号进行转义，否则就会提示语法错误 

6. expr 5 * 6    // 输出错误 

7. expr 5 * 6   // 输出30 
```



- **截取子字符串**

```
1. #简单截取:从字符串第 1 个字符开始往后截取 10 个字符 

2. str="SnailClimb is a great man" 

3. echo ${str:0:10} #输出:SnailClimb 

4.  

5. #根据表达式截取 

6. var="http://www.runoob.com/linux/linux-shell-variable.html" 

7.  

8. s1=${var%%t*}#h 

9. s2=${var%t*}#http://www.runoob.com/linux/linux-shell-variable.h 

10. s3=${var%%.*}#http://www 

11. s4=${var#*/}#/www.runoob.com/linux/linux-shell-variable.html 

12. s5=${var##*/}#linux-shell-variable.html 
```



#### 5. shell数组

bash支持一维数组（不支持多维数组），并且没有限定数组的大小。

```
1. #!/bin/bash 

2. array=(1 2 3 4 5); 

3. # 获取数组长度 

4. length=${#array[@]} 

5. # 或者 

6. length2=${#array[*]} 

7. #输出数组长度 

8. echo $length #输出：5 

9. echo $length2 #输出：5 

10. # 输出数组第三个元素 

11. echo ${array[2]} #输出：3 

12. unset array[1]# 删除下标为1的元素也就是删除第二个元素 

13. for i in ${array[@]};do echo $i ;done # 遍历数组，输出： 1 3 4 5  

14. unset arr_number; # 删除数组中的所有元素 

15. for i in ${array[@]};do echo $i ;done # 遍历数组，数组元素为空，没有任何输出内容 
```



#### 6. shell基本运算符

- 算法运算符

![img](http://pcc.huitogo.club/948d5f41c05827d90aeaff58c51a3d22)



例如简单的加法

```
1. #!/bin/bash 

2. a=3;b=3; 

3. val=`expr $a + $b` 

4. #输出：Total value : 6 

5. echo "Total value : $val" 
```



- 关系运算符

关系运算符只支持数字，不支持字符串，除非字符串的值是数字。

![img](http://pcc.huitogo.club/1e6b35cc096a502360320acbd56a11c8)



例如下面shell程序的作用是当score=100的时候输出A否则输出B

```
1. #!/bin/bash 

2. score=90; 

3. maxscore=100; 

4. if [ $score -eq $maxscore ] 

5. then 

6.  echo "A" 

7. else 

8.  echo "B" 

9. fi 
```



- 逻辑运算符

![img](http://pcc.huitogo.club/118b1bf269bdfe6019c031f2d885754d)



示例

```
1. #!/bin/bash 

2. a=$(( 1 && 0)) 

3. # 输出：0；逻辑与运算只有相与的两边都是1，与的结果才是1；否则与的结果是0 

4. echo $a; 
```



- 布尔运算符

![img](http://pcc.huitogo.club/6c090a66d295ced36f5632f36339be0d)



- 字符串运算符

![img](http://pcc.huitogo.club/afebf6cd30997125c7760bde8dc57f9a)



示例

```
1. #!/bin/bash 

2. a="abc"; 

3. b="efg"; 

4. if [ $a = $b ] 

5. then 

6.  echo "a 等于 b" 

7. else 

8.  echo "a 不等于 b" 

9. fi 
```



- 文件相关运算符

![img](http://pcc.huitogo.club/0a0a0cdb219065254211a7bd3333133c)



使用方式很简单，比如我们定义好了一个文件路径file="/usr/learnshell/test.sh" 如果我们想判断这个文件是否可读，可以这样if [ -r $file ] 如果想判断这个文件是否可写，可以这样-w $file，是不是很简单。



#### 7. shell流程控制

- if条件语句

简单示例

```
1. #!/bin/bash 

2. a=3; 

3. b=9; 

4. if [ $a -eq $b ] 

5. then 

6.  echo "a 等于 b" 

7. elif [ $a -gt $b ] 

8. then 

9.  echo "a 大于 b" 

10. else 

11.  echo "a 小于 b" 

12. fi 
```

需要注意的是，不同于我们常见的 Java 以及 PHP 中的 if 条件语句，shell if 条件语句中不能包含空语句也就是什么都不做的语句。



- for循环语句

举三个简单示例说明

1）输出当前列表中的数据

```
1. for loop in 1 2 3 4 5 

2. do 

3.   echo "The value is: $loop" 

4. done 
```



2）产生 10 个随机数

```
1. #!/bin/bash 

2. for i in {0..9}; 

3. do  

4.  echo $RANDOM; 

5. done 
```



3）输出1到5

```
1. #!/bin/bash 

2. for((i=1;i<=5;i++));do 

3.   echo $i; 

4. done; 
```



- while语句

1）基本的 while 循环语句

```
1. #!/bin/bash 

2. int=1 

3. while(( $int<=5 )) 

4. do 

5.   echo $int 

6.   let "int++" 

7. done 
```



2）while循环可用于读取键盘信息

```
1. echo '按下 <CTRL-D> 退出' 

2. echo -n '输入你最喜欢的电影: ' 

3. while read FILM 

4. do 

5.   echo "是的！$FILM 是一个好电影" 

6. done 
```



3）无限循环

```
1. while true 

2. do 

3.   command 

4. done 
```



#### 8. shell函数

- 不带参数没有返回值的函数

```
1. #!/bin/bash 

2. hello(){ 

3.   echo "这是我的第一个 shell 函数!" 

4. } 

5. echo "-----函数开始执行-----" 

6. hello 

7. echo "-----函数执行完毕-----" 
```



- 有返回值的函数

```
1. #!/bin/bash 

2. funWithReturn(){ 

3.   echo "输入第一个数字: " 

4.   read aNum 

5.   echo "输入第二个数字: " 

6.   read anotherNum 

7.   echo "两个数字分别为 $aNum 和 $anotherNum !" 

8.   return $(($aNum+$anotherNum)) 

9. } 

10. funWithReturn 

11. echo "输入的两个数字之和为 $?" 
```



- 带参数的函数

```
1. #!/bin/bash 

2. funWithParam(){ 

3.   echo "第一个参数为 $1 !" 

4.   echo "第二个参数为 $2 !" 

5.   echo "第十个参数为 $10 !" 

6.   echo "第十个参数为 ${10} !" 

7.   echo "第十一个参数为 ${11} !" 

8.   echo "参数总数有 $# 个!" 

9.   echo "作为一个字符串输出所有参数 $* !" 

10. } 

11. funWithParam 1 2 3 4 5 6 7 8 9 34 73 
```



#### 9. shell其他应用

下面举个我开发中写的一个用于项目服务更新的脚本

流程是查询当前服务有没有服务有没有开启 --> 关闭旧服务 --> 备份旧服务到指定位置并删除原位置 --> 将新服务移动到指定位置 --> 开启新服务



shell脚本示例如下：

```
1. #生产包目录  

3. THIS_JAR_INSTANCE='/home/ssc'  

5. #旧包名称  

7. THIS_OlDJAR_MODDULE='business-flow-1.0.1-SNAPSHOT.jar'  

9. #新包名称  

11. THIS_NEWJAR_MODDULE='business-flow-1.0.3-SNAPSHOT.jar'  

13. #备份操作  

15. cp ${THIS_JAR_INSTANCE}/${THIS_OlDJAR_MODDULE} /shengji/bak/20190227  

17. #记录服务进程id  

19. jar_pid=`ps -ef|grep -v grep | grep ${THIS_OlDJAR_MODDULE} |awk '{ print $2 }'`  

20.  

21. #判断服务进程id是否存在 若存在停止进程 并更新新包重启  若不存在 更新新包重启  

23. if [ ! -n $jar_pid ]; then  

25.   echo 'will-redpoly'  

27.  rm -f ${THIS_JAR_INSTANCE}/${THIS_OlDJAR_MODDULE}  

29.  cp ${THIS_NEWJAR_MODDULE} ${THIS_JAR_INSTANCE}  

31.  cd ${THIS_JAR_INSTANCE}  

33.  nohup java -jar ${THIS_NEWJAR_MODDULE} &  

35.  echo 'redpoly success0'  

37. else  

39.   kill -9 $jar_pid  

41.  echo 'kill' $jar_pid  

43.  rm -f ${THIS_JAR_INSTANCE}/${THIS_OlDJAR_MODDULE}  

45.  cp ${THIS_NEWJAR_MODDULE} ${THIS_JAR_INSTANCE}  

47.  cd ${THIS_JAR_INSTANCE}  

49.  nohup java -jar ${THIS_NEWJAR_MODDULE} &  

51.  echo 'redpoly success1'  

53. fi  
```