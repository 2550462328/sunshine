#### 1. 什么是OAuth2.0?

OAuth 是一个行业的标准授权协议，主要用来授权第三方应用获取有限的权限。而 OAuth 2.0是对 OAuth 1.0 的完全重新设计。

实际上它就是一种授权机制，它的最终目的是为第三方应用颁发一个有时效性的令牌 token，使得第三方应用能够通过该令牌获取相关的资源。



OAuth 2.0 比较常用的场景就是第三方登录，当你的网站接入了第三方登录的时候一般就是使用的 OAuth 2.0 协议。

另外，现在OAuth 2.0也常见于支付场景（微信支付、支付宝支付）和开发平台（微信开放平台、阿里开放平台等等）。



举个例子：

就是我的外卖到了小区门口，但他没有门卡进不来，但是物业又不可能给他门卡，有一种解决方案就是我到门口给他开门，但不能他每次来我都下来给他开门吧，于是物业经过我的同意给了它一张临时卡，可以在一段时期内进出小区给我送外卖。



我们把上面的例子搬到互联网，就是 OAuth 的设计了。

首先，居民小区就是储存用户数据的网络服务。比如，微信储存了我的好友信息，获取这些信息，就必须经过微信的"门禁系统"。

其次，快递员（或者说快递公司）就是第三方应用，想要穿过门禁系统，进入小区。

最后，我就是用户本人，同意授权第三方应用进入小区，获取我的数据。



简单说，**OAuth 就是一种授权机制。数据的所有者告诉系统，同意授权第三方应用进入系统，获取这些数据。系统从而产生一个短期的进入令牌（token），用来代替密码，供第三方应用使用。**



#### 2. 使用OAuth2.0有什么好处？

换句话说为什么不直接给外卖员一个永久卡，而是一张临时卡呢？



永久卡就是“密码”，临时卡就是“令牌”

令牌（token）与密码（password）的作用是一样的，都可以进入系统，但是有三点差异：

1. 令牌是短期的，到期会自动失效，用户自己无法修改。密码一般长期有效，用户不修改，就不会发生变化。
2. 令牌可以被数据所有者撤销，会立即失效。以上例而言，屋主可以随时取消快递员的令牌。密码一般不允许被他人撤销。
3. 令牌有权限范围（scope），比如只能进小区的二号门。对于网络服务来说，只读令牌就比读写令牌更安全。密码一般是完整权限。



上面这些设计，保证了令牌既可以让第三方应用获得权限，同时又随时可控，不会危及系统安全。这就是 OAuth 2.0 的优点。

注意，只要知道了令牌，就能进入系统。系统一般不会再次确认身份，所以**令牌必须保密，泄漏令牌与泄漏密码的后果是一样的**。 这也是为什么令牌的有效期，一般都设置得很短的原因。



#### 3. OAuth得4种授权类型？

- 授权码（authorization-code）
- 隐藏式（implicit）
- 密码式（password）：
- 客户端凭证（client credentials）



##### 3.1 授权码

授权码（authorization code）方式，指的是第三方应用先申请一个授权码，然后再用该码获取令牌。

这种方式是最常用的流程，安全性也最高，它适用于那些有后端的 Web 应用。授权码通过前端传送，令牌则是储存在后端，而且所有与资源服务器的通信都在后端完成。这样的前后端分离，可以避免令牌泄漏。



![img](http://pcc.huitogo.club/16de0930348bf00ea69e4a2962e79f82)



向第三方请求示例：



先申请授权码

```
1. https://b.com/oauth/authorize? 

2.  response_type=code& 

3.  client_id=CLIENT_ID& 

4.  redirect_uri=CALLBACK_URL& 

5.  scope=read 
```

用户跳转后，B 网站会要求用户登录，然后询问是否同意给予 A 网站授权。用户表示同意，这时 B 网站就会跳回redirect_uri参数指定的网址。跳转时，会传回一个授权码



A 网站拿到授权码以后，就可以在后端，向 B 网站请求令牌。

```
1. https://b.com/oauth/token? 

2. client_id=CLIENT_ID& 

3. client_secret=CLIENT_SECRET& 

4. grant_type=authorization_code& 

5. code=AUTHORIZATION_CODE& 

6. redirect_uri=CALLBACK_URL
```

B 网站收到请求以后，就会颁发令牌。具体做法是向redirect_uri指定的网址，发送一段 JSON 数据。



##### 3.2 隐藏式

有些 Web 应用是纯前端应用，没有后端。这时就不能用上面的方式了，必须将令牌储存在前端。RFC 6749 就规定了第二种方式，允许直接向前端颁发令牌。这种方式没有授权码这个中间步骤，所以称为（授权码）"隐藏式"（implicit）。



![img](http://pcc.huitogo.club/16cce30d39b8ff3bd6264770d6fdb320)



请求示例：

```
1. https://b.com/oauth/authorize? 

2.  response_type=token& 

3.  client_id=CLIENT_ID& 

4.  redirect_uri=CALLBACK_URL& 

5.  scope=read 
```

用户跳转到 B 网站，登录后同意给予 A 网站授权。这时，B 网站就会跳回redirect_uri参数指定的跳转网址，并且把令牌作为 URL 参数，传给 A 网站。



##### 3.3 密码式

如果你高度信任某个应用，RFC 6749 也允许用户把用户名和密码，直接告诉该应用。该应用就使用你的密码，申请令牌，这种方式称为"密码式"（password）。



请求示例：

```
1. https://oauth.b.com/token? 

2.  grant_type=password& 

3.  username=USERNAME& 

4.  password=PASSWORD& 

5.  client_id=CLIENT_ID 
```

B 网站验证身份通过后，直接给出令牌。注意，这时不需要跳转，而是把令牌放在 JSON 数据里面，作为 HTTP 回应。



##### 3.4 客户端凭证

适用于没有前端的命令行应用，即在命令行下请求令牌。



请求示例

```
1. https://oauth.b.com/token? 

2.  grant_type=client_credentials& 

3.  client_id=CLIENT_ID& 

4.  client_secret=CLIENT_SECRET 
```

B 网站验证通过以后，直接返回令牌。



#### 4. 令牌使用

A 网站拿到令牌以后，就可以向 B 网站的 API 请求数据了。

此时，每个发到 API 的请求，都必须带有令牌。具体做法是在请求的头信息，加上一个Authorization字段，令牌就放在这个字段里面。

```
1. curl -H "Authorization: Bearer ACCESS_TOKEN"  "https://api.b.com" 
```

上面命令中，ACCESS_TOKEN就是拿到的令牌。



#### 5. 更新令牌

令牌的有效期到了，如果让用户重新走一遍上面的流程，再申请一个新的令牌，很可能体验不好，而且也没有必要。OAuth 2.0 允许用户自动更新令牌。

具体方法是，B 网站颁发令牌的时候，一次性颁发两个令牌，一个用于获取数据，另一个用于获取新的令牌（refresh token 字段）。令牌到期前，用户使用 refresh token 发一个请求，去更新令牌。



请求示例

```
1. https://b.com/oauth/token? 

2.  grant_type=refresh_token& 

3.  client_id=CLIENT_ID& 

4.  client_secret=CLIENT_SECRET& 

5.  refresh_token=REFRESH_TOKEN 
```

B 网站验证通过以后，就会颁发新的令牌。



#### 6. github第三方登录实例

A 网站允许 GitHub 登录，背后就是下面的流程

```
1. A 网站让用户跳转到 GitHub。 

2. GitHub 要求用户登录，然后询问"A 网站要求获得 xx 权限，你是否同意？" 

3. 用户同意，GitHub 就会重定向回 A 网站，同时发回一个授权码。 

4. A 网站使用授权码，向 GitHub 请求令牌。 

5. GitHub 返回令牌. 

6. A 网站使用令牌，向 GitHub 请求用户数据
```



首先需要登记应用，设置授权回调地址

通过https://github.com/settings/applications/new 这个页面去登记

提交表单以后，GitHub 会返回客户端 ID（client ID）和客户端密钥（client secret），这就是应用的身份识别码。



简单的登录页

```
1. Login with GitHub 

2.  

3. <script> 

4.   var client_id = 'ca0d972cabea7c246eb3'; 

5.   var authorize_uri = 'https://github.com/login/oauth/authorize'; 

6.   var redirect_uri = 'http://localhost:8082/oauth/redirect'; 

7.   var link = document.getElementById('login'); 

8.   link.href = authorize_uri + '?client_id=' + client_id + '&redirect_uri=' + redirect_uri + "&scope=user"; 

9. </script> 
```



授权成功后的回调逻辑（获取授权码+ 获取token + 获取用户信息）

```
1.   @GetMapping("/redirect") 

2.   public ModelAndView redirectUrl(@RequestParam(name = "code",required = false) String code) { 

3.  

4.     ModelAndView mv = new ModelAndView(); 

5.  

6.     log.info("获取github官网用户授权码{}", code); 

7.  

8.     String getAccessUrl = "https://github.com/login/oauth/access_token?client_id=" 

9.         + clientid + "&client_secret=" 

10.         + clientsecret + "&code=" + code; 

11.  

12.     HttpHeaders headers = new HttpHeaders(); 

13.     List<MediaType> mediaTypes = new ArrayList<>(); 

14.     mediaTypes.add(MediaType.APPLICATION_JSON); 

15.     headers.setAccept(mediaTypes); 

16.  

17.     String access = httpClient.send(getAccessUrl,HttpMethod.POST,null,headers); 

18.     JSONObject accessObj = JSONObject.parseObject(access); 

19.  

20.     String accessToken = accessObj.getString("access_token"); 

21.  

22.     log.info("获取github官网用户登录令牌{}", accessToken); 

23.  

24.     // 第一种方式 

25. //    String getUserUrl = "https://api.github.com/user?access_token="+accessToken; 

26.  

27.     String getUserUrl = "https://api.github.com/user"; 

28.  

29.     HttpHeaders headers_getUser = new HttpHeaders(); 

30.     List<MediaType> mediaTypes_getUser = new ArrayList<>(); 

31.     mediaTypes_getUser.add(MediaType.APPLICATION_JSON); 

32.  

33.     headers_getUser.setAccept(mediaTypes_getUser); 

34.     // 第二种方式 

35.     headers_getUser.set("Authorization","token "+ accessToken); 

36.  

37.     String user = httpClient.send(getUserUrl,HttpMethod.GET,null,headers_getUser); 

38.  

39.     log.info("获取github官网用户信息{}", user); 

40.  

41.     JSONObject userInfo = JSONObject.parseObject(user); 

42.  

43.     mv.addObject("userName", userInfo.getString("login")); 

44.     mv.setViewName("/oauth/success"); 

45.     return mv; 

46.   } 
```

到此github的第三方登录就完成了，其他qq和weixin啥的逻辑类似。