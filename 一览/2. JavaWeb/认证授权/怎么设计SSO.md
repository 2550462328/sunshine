#### 1. 什么是SSO?

SSO(Single Sign On)即单点登录说的是用户登陆多个子系统的其中一个就有权访问与其相关的其他系统。举个例子我们在登陆了京东金融之后，我们同时也成功登陆京东的京东超市、京东家电等子系统。



#### 2. 单点登录的好处？

- 用户角度 :用户能够做到一次登录多次使用，无需记录多套用户名和密码，省心。

- 系统管理员角度 : 管理员只需维护好一个统一的账号中心就可以了，方便。
- 新系统开发角度: 新系统开发时只需直接对接统一的账号中心即可，简化开发流程，省时。




#### 3. 如何设计一个SSO系统

总体架构流程如下

![img](http://pcc.huitogo.club/b1c9802cffaae269779dac35ac3f3501)



以上过程



![img](http://pcc.huitogo.club/3135f2b7ef186f6b282cb9dffdd0dc5f)



具体细节讨论如下



##### 3.1 用户登录状态的存储与校验

常见的Web框架对于Session的实现都是生成一个SessionId存储在浏览器Cookie中。然后将Session内容存储在服务器端内存中，这个 ken.io 在之前Session工作原理中也提到过。整体也是借鉴这个思路。



用户登录成功之后，生成AuthToken交给客户端保存。如果是浏览器，就保存在Cookie中。如果是手机App就保存在App本地缓存中。用户在浏览需要登录的页面时，客户端将AuthToken提交给SSO服务校验登录状态/获取用户登录信息。



对于登录信息的存储，建议采用Redis，使用Redis集群来存储登录信息，既可以保证高可用，又可以线性扩充。



![img](http://pcc.huitogo.club/c7b3439dff12b2d9d27af569309c7fdb)



##### 3.2 用户登录/登录校验

登录时序图

![img](http://pcc.huitogo.club/80336f2c4043c014f4b5a826b89c6885)



按照上图，用户登录后Authtoken保存在Cookie中。 domian= test. com 浏览器会将domain设置成 .test.com， 这样访问所有*.test.com的web站点，都会将Authtoken携带到服务器端。 然后通过SSO服务，完成对用户状态的校验/用户登录信息的获取



校验时序图

![img](http://pcc.huitogo.club/0992a7ebcb0be60dfb62c66900703ae4)



##### 3.3 用户登出

用户登出时要做的事情很简单：

- 服务端清除缓存（Redis）中的登录状态

- 客户端清除存储的AuthToken




登出时序图

![img](http://pcc.huitogo.club/ff59366d81f10711bf8c8b9f48971550)



##### 3.4 跨域登录、登出

前面提到过，核心思路是客户端存储AuthToken，服务器端通过Redis存储登录信息。由于客户端是将AuthToken存储在Cookie中的。所以跨域要解决的问题，就是如何解决Cookie的跨域读写问题。



解决跨域的核心思路就是：

1. 登录完成之后通过回调的方式，将AuthToken传递给主域名之外的站点，该站点自行将AuthToken保存在当前域下的Cookie中。

2. 登出完成之后通过回调的方式，调用非主域名站点的登出页面，完成设置Cookie中的AuthToken过期的操作。

3. 跨域登录（主域名已登录）




跨域登录时序图

![img](http://pcc.huitogo.club/40f1566240f7c22c05d17296de17fed4)



跨域登出时序图

![img](http://pcc.huitogo.club/e117b5636848b304755edd96a8ad7a40)