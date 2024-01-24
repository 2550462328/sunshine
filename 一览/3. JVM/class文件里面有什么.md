class文件是由java类编译成的文件，也就是我们经常说的字节码，是一组以8位字节为基础单位的二进制流，字节码再由jvm转换成机器码（二进制）进行执行。

class文件也**是java语言跨平台的根本**，比如在windows系统下编译出来的class文件放到linux下也可以执行。同时也**是不同语言在java虚拟机之间的重要桥梁**，比如以下语言都是会编译成class文件放到jvm中去执行：



![img](http://pcc.huitogo.club/f1030e9779a2b62ac20b71586a389527)



**class文件怎么生成？**

一般编译器如eclipse和idea在build项目的时候都会生成class文件，针对单个文件我们可以通过javac java类名称去执行

将class文件反编译成java类可以通过javap、jad指令或者jd-gui工具，区别在于jad反编译出来的更接近于jvm执行的格式，javap反编译出来的更接近于机器执行的格式，而jd-gui则努力还原程序本来的代码。



例如源码是：

```
1.  public class Test {  

2.      public static void main(String[] args) {  

3.          String s = "a";  

4.          s += "b";  

5.      }  

6.  }  
```



编译成class文件后，再分别由三个进行还原

javap -v Test.class

![img](http://pcc.huitogo.club/eeb50a501a699c00b2ec749b9b293956)



可以看到这里大量aload、invoke和astore操作，类似于CPU的工作步骤了。

![img](http://pcc.huitogo.club/6f7ffda72243ef483cb5b36634bcad98)



jad Test.class

![img](http://pcc.huitogo.club/db22eb631500ccd33a073ce10afb69ce)

这里可以看到不仅加上了默认构造函数，而且对于 +=操作，在jvm上是使用StringBuilder来实现的。所以想要看你的代码被jvm怎么理解和执行的，可以使用jad指令来反编译一下。



在jd-gui工具中打开Test.class

![img](http://pcc.huitogo.club/755787d907291ba07d87322b402994d4)



可以看出基本和源码不差



**class文件有什么？**

class的结构不像XML等描述语言，由于它没有任何分隔符号。所以在其中的数据项，无论是字节顺序还是数量，都是被严格限定的，哪个字节代表什么含义，长度是多少，先后顺序如何，都不允许改变。

![img](http://pcc.huitogo.club/eed87605b3de2282cbd191daad162b69)



根据 Java 虚拟机规范，类文件由单个 ClassFile 结构组成：

```
1.  ClassFile {  

2.      u4             magic; //Class 文件的标志  

3.      u2             minor_version;//Class 的小版本号  

4.      u2             major_version;//Class 的大版本号  

5.      u2             constant_pool_count;//常量池的数量  

6.      cp_info        constant_pool[constant_pool_count-1];//常量池  

7.      u2             access_flags;//Class 的访问标记  

8.      u2             this_class;//当前类  

9.      u2             super_class;//父类  

10.     u2             interfaces_count;//接口  

11.     u2             interfaces[interfaces_count];//一个类可以实现多个接口  

12.     u2             fields_count;//Class 文件的字段属性  

13.     field_info     fields[fields_count];//一个类会可以有个字段  

14.     u2             methods_count;//Class 文件的方法数量  

15.     method_info    methods[methods_count];//一个类可以有个多个方法  

16.     u2             attributes_count;//此类的属性表中的属性数  

17.     attribute_info attributes[attributes_count];//属性表集合  

18. }  
```



上述变成Class文件字节码结构组织示意图如下：

![img](http://pcc.huitogo.club/4226c6182136baf025db13463f4e5c3a)

基本上魔数、主副版本号、访问标识、父类索引（除了Object类外）和类索引是每个类都必须的，其他常量池、接口、方法、字段和属性都是因类而异的。



#### 1. 魔数

每个 Class 文件的头四个字节称为魔数（Magic Number）,它的唯一作用是**确定这个文件是否为一个能被虚拟机接收的 Class 文件。**

程序设计者很多时候都喜欢用一些特殊的数字表示固定的文件类型或者其它特殊的含义。



#### 2. 主副版本号

紧接着魔数的四个字节存储的是 Class文件的版本号：**第五和第六是次版本号**，**第七和第八是主版本号**。

高版本的 Java 虚拟机可以执行低版本编译器生成的 Class 文件，但是低版本的 Java虚拟机不能执行高版本编译器生成的 Class 文件。所以，我们在实际开发的时候要确保开发的的 JDK 版本和生产环境的 JDK版本保持一致。



#### 3. 常量池

紧接着主次版本号之后的是常量池，常量池的数量是constant_pool_count-1（**常量池计数器是从1开始计数的，将第0项常量空出来是有特殊考虑的，索引值为0代表“不引用任何一个常量池项”**）。

常量池主要存放两大常量：字面量和符号引用。字面量比较接近于 Java语言层面的的常量概念，如文本字符串、声明为 final 的常量值等。而符号引用则属于编译原理方面的概念。包括下面三类常量：

- 类和接口的全限定名
- 字段的名称和描述符
- 方法的名称和描述符



常量池中每一项常量都是一个表，这14种表有一个共同的特点：**开始的第一位是一个 u1类型的标志位 -tag 来标识常量的类型，代表当前这个常量属于哪种常量类型**

![img](http://pcc.huitogo.club/37b6d9413d652e619f5c186ac80aa0e4)



#### 4. 访问标识

在常量池结束之后，紧接着的两个字节代表访问标志，这个标志用于识别一些类或者接口层次的访问信息，包括：这个 Class 是类还是接口，类的访问权限等。



其中有以下类型：

![img](http://pcc.huitogo.club/6ab68b7418c87a11490d1090050541e9)



#### 5. 类索引和父类索引

类索引用于确定这个类的全限定名，父类索引用于确定这个类的父类的全限定名，由于Java 语言的单继承，所以父类索引只有一个，除了 java.lang.Object 之外，所有的 java类都有父类，因此除了 java.lang.Object 外，所有 Java 类的父类索引都不为 0。



#### 6. 接口信息

包括当前类实现的接口数量和接口集合，如果当前是接口的话，则这里放置的是extends的接口数量和接口集合。



#### 7. 字段信息

包括当前类中的字段数量和字段集合，不包括方法中的局部变量。



字段集合中的字段表示如下：

![img](http://pcc.huitogo.club/ce0695c037e5b7108e5d9ca98c0091b7)

- access_flags：字段的作用域
- name_index：对常量池的引用，表示的字段的名称；
- descriptor_index:：对常量池的引用，表示字段和方法的描述符；
- attributes_count：一个字段还会拥有一些额外的属性，attributes_count存放属性的个数；
- attributes[attributes_count]：存放具体属性具体内容。

在上述信息中，对于字段作用域可以用标识位来表示（类比枚举），而字段的名称和类型只能引用常量池中的常量来表示，所以这里只会存放索引。



对于access_flags的取值有：

![img](http://pcc.huitogo.club/f66cd83ba117581a094ba9af6da62c56)



#### 8. 方法信息

包括类中的方法数量和方法集合。

方法集合中对方法的表示和字段一样

![img](http://pcc.huitogo.club/03574480770f89508e9ff267193fbb8c)



其中access_flags的取值有：

![img](http://pcc.huitogo.club/9009549472979936d7b3775b4674fa79)



#### 9. 属性信息

在 Class文件，字段表，方法表中都可以携带自己的属性表集合，以用于描述某些场景专有的信息。与Class文件中其它的数据项目要求的顺序、长度和内容不同，属性表集合的限制稍微宽松一些，不再要求各个属性表具有严格的顺序，并且只要不与已有的属性名重复，任何人实现的编译器都可以向属性表中写入自己定义的属性信息，Java 虚拟机运行时会忽略掉它不认识的属性。