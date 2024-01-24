除了通俗的@Getter、@Setter、@Data、@slf4j等等常用外介绍几个实用性高的



**@Accessor(chain = true)**

譬如

```
1.  @Accessors(chain = true)  

2.  @Setter  

3.  @Getter  

4.  public class Student {  

5.      private String name;  

6.      private int age;  

7.  }  
```



可以实现传统的链式方法

```
1.  public class Student {  

2.      private String name;  

3.      private int age;  

4.      // 忽略getter方法      

5.      // set完之后会返回自身用于链式操作  

6.      public Student setName(String name) {  

7.          this.name = name;  

8.          return this;  

9.      }  

10.     public Student setAge(int age) {  

11.         return this;  

12.     }  

13. }  
```



**@RequiredArgsConstructor(staticName = "ofName")**

这个用来实现静态构造方法

首先什么是静态构造方法，比如说新建一个ArrayList对象

```
1.  // 传统构造方法  

2.  List<String> list = new ArrayList<>();  

3.  // 静态构造方法  

4.  List<String> list = Lists.newArrayList();  
```

后者无疑更专业，语义更强



使用lombok实现的话就是这样

```
1.  @Setter  

2.  @Getter  

3.  @RequiredArgsConstructor(staticName = "of")  

4.  public class Student {  

5.      @NonNull private String name;  

6.      private int age;  

7.  }  
```



具体使用可以如下

```
1.  // 语义更强  

2.  Student student = Student.of("zhanghui");  
```



**@Builder**

对于对象的builder模式相信大家在effective java中都见识过，出发点是好的，就是代码要不要太繁重了

之前使用builder模式如下

```
1.  public class Student {  

2.      // 忽略name和age属性和getter/setter  

3.      public static Builder builder(){  

4.              return new Builder();  

5.      }  

6.      public static class Builder{  

7.              private String name;  

8.              private int age;  

9.              public Builder name(String name){  

10.                     this.name = name;  

11.                     return this;  

12.             }  

13.             public Builder age(int age){  

14.                     this.age = age;  

15.                     return this;  

16.             }  

17.             public Student build(){  

18.                     Student student = new Student();  

19.                     student.setAge(age);  

20.                     student.setName(name);  

21.                     return student;  

22.             }  

23.     }  

24. }  
```



使用lombok如下

```
1.  @Builder  

2.  public class Student {  

3.      private String name;  

4.      private int age;  

5.  }  
```



**@SneakyThrows**

@SneakyThrows的用法比较简单，其实就是对于异常的一个整理，将checked exception 看做unchecked exception， 不处理，直接扔掉。 减少了到处写catch的不便利性。



**@Delegate**

比如说我们希望对RestTemplate的方法进行修改，我们新建类对它进行包装，

```
1.  public class ExtractRestTemplate implements RestOperations {  

2.      // 为什么不直接extends ResTemplate，我觉得那样得话就调用不到 ResTemplate的非继承方法了 还有就是组合 > 继承  

3.      // 这样直接拿个壳将它包装起来，既可以使用它原有方法，又可以添加新功能  

4.      // 包装器模式 + 组合思想  

5.      private RestTemplate restTemplate;  

6.      public ExtractRestTemplate(RestTemplate restTemplate) {  

7.              this.restTemplate = restTemplate;  

8.      }  

9.      // 可以加上自定义的方法结合restTemplate替代原有的RestTemplate的方法  

10.     // 这里必须实现RestOperation中的所有方法  

11. }  
```

这里因为RestOperations是接口，所以必须实现它的抽象方法，所以八八八八一大堆没用到的代码诞生了。



使用lombok后

```
1.  @AllArgsConstructor  

2.  public abstract class ExtractRestTemplate implements RestOperations {  

3.      @Delegate  

4.      protected volatile RestTemplate restTemplate;  

5.  }  
```