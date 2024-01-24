#### 1. 什么是序列化？

Serialization（序列化）是一种将对象以一连串的字节描述的过程；反序列化deserialization是一种将这些字节重建成一个对象的过程。java序列化API提供一种处理对象序列化的标准机制。



#### 2. 序列化过程是什么？

1. 将对象实例相关的类元数据输出。
2. 当前类序列化终止，开始输出实际数据值。
3. 类元数据完了以后，对象实例类的超类元数据输出。
4. 超类序列化终止，开始输出实际数据值。



#### 3. 序列化文件内容

已知java对象如下：

```
1.  @Data  

2.  class SerializeUser implements Serializable{  

3.      private static final long serialVersionUID = 1L;  

4.      private String name = "zhanghui";  

5.      private int password = 123456;  

6.      private SerializeParent parent = new SerializeParent();  

7.  }  


8.  @Data  

9.  class SerializeParent implements Serializable{  

10.     private static final long serialVersionUID = 1L;  

11.     private String pversion;  

12. } 
```



序列化后

![img](http://pcc.huitogo.club/04a5741c0a52ca590d4cc2a42ab6a193)



在上文中，按照之前分析的序列化算法的步骤来看序列化后的内容（ASII码）

AC ED ：根据协议文档的约定,由于所有二进制流(文件)都需要在流的头部约定魔数(magic number=STREAM_MAGIC),即java object 序列化后的流的魔数约定为ACED;

00 05：流协议版本是short类型(2字节)并且值为5,则十六进制值为0005;

73 72：java byte型长度为1字节,所以0x73 0x72直接对应到字节流上，0x73(TC_OBJECT)代表这是一个对象,0x72(TC_CLASSDESC)代表这个之后开始是对类的描述。

00 0D：类名的长度

53 65 72 69 61 6C 69 7A 69 55 73 65 72：类名，也就是SerializeUser

00 00 00 00 00 00 00 00 01：前面8个00对应long定义的8字节，后面的01对应的1L，这里就是描述序列化serialVersionUID

02：根据classDescFlags定义

```
final static byte SC_SERIALIZABLE = 0x02; // 02代表可序列化
```

00 03：这两个字节代表类中域的个数，这里是3个

49：代表“I”，也就是int

00 08：描述字段的长度，这里就是password的长度

70 61 73 73 77 6F 72 64：八个字节表示的就是password

4C：代表“L”，也就是String

00 04：描述字段的长度，这里就是name的长度

6E 61 6D 65：四个字节表示的就是name

74：TC_STRING. 代表一个new String，用String来引用对象

00 12：引用对象String的长度

4C 6A 61 76 61 2F 6C 61 6E 67 2F 53 74 75 69 6E 3B：这一段经ACII翻译出来就是Ljava/lang/String; 也就是java的String类的签名，这里指的就是Strina name = “zhanghui”，后面那个“zhanghui”所在的String类的引用

4C：域的类型

00 06：域的长度，这里的域指的是parent的长度

70 61 72 65 6E 74：六个字节，指的就是parent

74：TC_STRING. 代表一个new String，用String来引用对象

00 11：该String的长度

4C 53 65 72 69 61 6C 69 7A 65 50 61 72 65 6E 74 3B：这一段就是SerializeParent类的jvm标准对象签名

78 70：根据协议

```
final static byte TC_ENDBLOCKDATA = (byte)0x78;   // 78代表该块定义的序列化流结束了  
final static byte TC_NULL = (byte)0x70;   // 70是NULL，代表没有超类了
```

00 01 E2 40：对password的赋值，password是int类型，4个字节

74 00 08 7A 68 61 6E 67 68 75 69：声明string类型和长度后对name的赋值



\#####################子类结束###父类雷同##################################



73 72 ：声明一个新的对象，这里开始描述父类SerializeParent

00 0F：SerializeParent类的长度

53 65 72 69 61 6C 69 7A 65 50 61 72 65 6E 74：这里描述的就是SerializeParent

00 00 00 00 00 00 00 00 01 02 00 01 ：这里跟SerializeUser类里描述的serialVersionUID一样

4C 00 08 70 76 65 72 73 69 6F 6E：描述的是pversion字段

71：表示TC_REFRENCE 引用

71 00 7E 00 01：表示id为9999后写入的对象指向了编号为1的对象，这里讲的就是序列化对象重复写入的问题，多个相同对象写入序列化文件的情况实际指向同一个对象，只不过在文件中加入了引用关系。这里定义的就是这么个规则。

78 70：标志着序列化的结束

70：NUll，表示pversion对象为空，没有赋值



#### 4. 序列化相关事项？

**Q1：什么时候serialVersionId 赋值默认值（1L），什么时候赋值为随机值？**

一般情况设置为1L可以解决序列化的问题了，设置成随机值有两种情况考虑

- 安全情况，在使用1L，有可能反序列化出来的对象不是自己想要的对象，于是序列化方和反序列化方就定义一个随机值，在序列化和反序列化中保证得到的就是自己定义的对象。
- 版本控制，比如在升级版本的情况下不希望定义的对象可以向下兼容，那么我可以更改对象的serialVersionId值，这样不会出现新老对象序列化冲突的情况。



**Q2：不希望对象中某些字段被序列化怎么办？**

- 给字段加上trasinent注释

- 给字段加上static修饰

  因为static修饰的字段代表着类的状态，而序列化和反序列化得出是对象在某一时刻的状态。

- 将不希望序列化的字段加入到父类中，父类不实现serializeable接口

  原理就是不实现serializeable接口的类不会被序列化



**Q3：有些字段我不希望明文传输怎么办？**

比如说密钥、密码之类的数据，我们可以自定义序列化传输内容，也就是在传输前自己处理一下



解决方案就是在对象中加入writeObject和readObject处理

如果被序列化的类中定义了writeObject 和 readObject 方法，虚拟机会试图调用对象类里的 writeObject 和 readObject 方法，进行用户自定义的序列化和反序列化。



示例如下：

```
1.  private void writeObject(ObjectOutputStream out){  

2.        PubField putFields = out.putFields();  

3.        password = encry(password);//模拟加密  

4.        putFields.put("password", password);  

5.        out.writeFields();  

6.  }  


7.  private void readObject(ObjectInputStream in){  

8.       GetField readFields = in.readFields();  

9.       Object object = readFields.get("password","");  

10.      password = deEncry(object);//模拟解密  

11. }  
```



**Q4：多次序列化写入相同对象，再一一取出来，得到的是不同对象还是相同对象？**

相同对象，序列化文件中只会有一个对象，重复写入的话只会新增引用。