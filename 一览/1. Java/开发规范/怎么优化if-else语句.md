**可以试试使用策略 + 简单工厂**

在写代码的时候经常遇到的一种业务场景是if-else  if -else

这种假设一种业务场景，对于不同用户的vip等级在付费商品的时候获得不同的折扣



在不加思索下，你可能会写下如下代码

```
1.  public BigDecimal calPrice(BigDecimal orderPrice, String buyerType) {  

2.      if (用户是专属会员) {  

3.          returen 7折价格;  

4.      }  

5.      if (用户是超级会员) {  

6.          return 8折价格;  

7.      }  

8.      if (用户是普通会员) {  

9.          return 9折价格;  

10.     }  

11.     return 原价;  

12. }  
```

这种还没有考虑在if里面可能还会有if的场景，可以想象只是一个思路写出来的代码就这么繁杂了，其实主要是可读性不高和维护难。



那么怎么优化？

首先将计算价格这个行为提取出来

```
1.  public interface UserPayService {  

2.      /** 

3.       * 计算应付价格 

4.       */  

5.      public BigDecimal quote(BigDecimal orderPrice);  

6.  }  
```



然后针对不同的会员情况进行具体实现

```
1.  public class ParticularlyVipPayService implements UserPayService {  

2.      @Override  

3.      public BigDecimal quote(BigDecimal orderPrice) {  

4.          return 7折价格;  

5.      }  

6.  }  

7.  public class SuperVipPayService implements UserPayService {  

8.      @Override  

9.      public BigDecimal quote(BigDecimal orderPrice) {  

10.         return 8折价格;  

11.     }  

12. }  

13. public class VipPayService implements UserPayService {  

14.     @Override  

15.     public BigDecimal quote(BigDecimal orderPrice) {  

16.         return 9折价格;  

17.     }  

18. }  
```



现在使用起来如下：

```
1.  public static void main(String[] args) {  

2.      UserPayService strategy = new VipPayService();  

3.      BigDecimal quote = strategy.quote(300);  

4.      System.out.println("普通会员商品的最终价格为：" + quote.doubleValue());  

5.      strategy = new SuperVipPayService();  

6.      quote = strategy.quote(300);  

7.      System.out.println("超级会员商品的最终价格为：" + quote.doubleValue());  

8.  }  
```



在Spring中因为具体实现的Service交由Spring托管的，所以使用起来如下

```
1.  public BigDecimal calPrice(BigDecimal orderPrice,User user) {  

2.       String vipType = user.getVipType();  

3.       if (vipType == 专属会员) {  

4.          //伪代码：从Spring中获取超级会员的策略对象  

5.          UserPayService strategy = Spring.getBean(ParticularlyVipPayService.class);  

6.          return strategy.quote(orderPrice);  

7.       }  

8.       if (vipType == 超级会员) {  

9.          UserPayService strategy = Spring.getBean(SuperVipPayService.class);

10.         return strategy.quote(orderPrice);  

11.      }  

12.      if (vipType == 普通会员) {  

13.         UserPayService strategy = Spring.getBean(VipPayService.class);  

14.         return strategy.quote(orderPrice);  

15.      }  

16.      return 原价;  

17. }  
```

这样看起来好像if个数没有减少噢~，但至少我们出发点是好的，哈哈哈哈



以上就是用基本的**策略模式**去优化了一下，但是策略模式有个很大的**弊端**，就是

客户端必须知道所有的策略类，并自行决定使用哪一个策略类。这就意味着客户端必须理解这些算法的区别，以便适时选择恰当的算法类。

换句话说就是我在给用户结算的时候我得清楚了解到每一个vip等级的计算方式。



**那可不可以隐藏这种行为呢？**

当然是可以的了，需要借助到另一个小伙伴**工厂模式，**优化的思路就是我只给你用户的vip等级，你自动给我对应的计算方式。而不需要我自己去找方法计算。



工厂模式的话我们先建造一座可以帮我们生产计算方式的工厂

```
1.  public class UserPayServiceStrategyFactory {  

2.      private static Map<String,UserPayService> services = new ConcurrentHashMap<String,UserPayService>();  

3.      public  static UserPayService getByUserType(String type){  

4.          return services.get(type);  

5.      }  

6.      public static void register(String userType,UserPayService userPayService){  

7.          Assert.notNull(userType,"userType can't be null");  

8.          services.put(userType,userPayService);  

9.      }  

10. }  
```



有了工厂再来计算用户真正支付的金额就很简单了，你只要告诉工厂用户的vip等级，工厂就给你一个计算方式，直接调用就可以了

```
1.  public class UserPayServiceStrategyFactory {  

2.      private static Map<String,UserPayService> services = new ConcurrentHashMap<String,UserPayService>();  

3.      public  static UserPayService getByUserType(String type){  

4.          return services.get(type);  

5.      }  

6.      public static void register(String userType,UserPayService userPayService){  

7.          Assert.notNull(userType,"userType can't be null");  

8.          services.put(userType,userPayService);  

9.      }  

10. }  
```



这样通过策略 + 工厂模式的优化，代码量和可读性大大优化了，但是还有个问题，我每次新增一个计算方式，怎么放到工厂中呢？



这里需要涉及到Spring的生命周期了

![img](http://pcc.huitogo.club/2c269631f633da39d566bfa20a49ed97)



每个新增的计算方式都是一个Service，会放到Spring中托管的，那么这个Bean肯定是遵循Spring中Bean的生命周期，所以我们不妨在这些**Bean初始化后将它们放到工厂中**

```
1.  @Service  

2.  public class ParticularlyVipPayService implements UserPayService,InitializingBean {  

3.      @Override  

4.      public BigDecimal quote(BigDecimal orderPrice) {  

5.         return 7折价格;  

6.      }  

7.      @Override  

8.      public void afterPropertiesSet() throws Exception {  

9.          UserPayServiceStrategyFactory.register("ParticularlyVip",this);  

10.     }  

11. }  

12. @Service  

13. public class SuperVipPayService implements UserPayService ,InitializingBean{  

14.     @Override  

15.     public BigDecimal quote(BigDecimal orderPrice) {  

16.         return 8折价格;  

17.     }  

18.     @Override  

19.     public void afterPropertiesSet() throws Exception {  

20.         UserPayServiceStrategyFactory.register("SuperVip",this);  

21.     }  

22. }  

23. @Service    

24. public class VipPayService implements UserPayService,InitializingBean {  

25.     @Override  

26.     public BigDecimal quote(BigDecimal orderPrice) {  

27.         return 9折价格;  

28.     }  

29.     @Override  

30.     public void afterPropertiesSet() throws Exception {  

31.         UserPayServiceStrategyFactory.register("Vip",this);  

32.     }  

33. }  
```

当然这里还有优化空间，比如register到工厂中的代码是重复代码，可以利用模板方法统一一下；



最后记一下我看到这篇文章触动的一句话，**对于设计模式的学习，重要的是学习其思想，而不是代码实现**！！！