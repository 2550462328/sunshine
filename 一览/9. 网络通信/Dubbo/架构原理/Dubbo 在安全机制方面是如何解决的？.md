通过令牌验证在**注册中心**控制权限，以决定要不要下发令牌给消费者，可以防止消费者绕过注册中心访问提供者。

另外通过注册中心可灵活改变授权方式，而不需修改或升级提供者。

![认证流程](https://pcc.huitogo.club/z0/20240312123123.png)



从源码角度看核心是通过TokenFilter进行Token检验和认证的，那我们看看Token是怎么生成、发送和验证的

#### 1.【服务提供者】随机 Token

在 ServiceConfig 的 `#doExportUrlsFor1Protocol(protocolConfig, registryURLs)` 方法中，随机生成 Token ：

```
if (!ConfigUtils.isEmpty(token)) {
    if (ConfigUtils.isDefault(token)) { // true || default 时，UUID 随机生成
        map.put("token", UUID.randomUUID().toString());
    } else {
        map.put("token", token);
    }
}
```



#### 2. 【服务消费者】接收 Token

服务**消费者**，从注册中心，获取服务提供者的 **URL** ，从而获得该服务着的 Token 。
所以，即使服务提供者随机生成 Token ，消费者一样可以拿到。



#### 3. 【服务消费者】发送 Token

RpcInvocation 在创建时，“**自动**”带上 Token ，如下图所示：

![RpcInvocation](https://pcc.huitogo.club/z0/202403124125123.png)



#### 4. 【服务提供者】认证 Token

`com.alibaba.dubbo.rpc.filter.TokenFilter` ，实现 Filter 接口，**令牌验证** Filter 实现类。代码如下：

```
@Activate(group = Constants.PROVIDER, value = Constants.TOKEN_KEY)
public class TokenFilter implements Filter {

    @Override
    public Result invoke(Invoker<?> invoker, Invocation inv) throws RpcException {
        // 获得服务提供者配置的 Token 值
        String token = invoker.getUrl().getParameter(Constants.TOKEN_KEY);
        if (ConfigUtils.isNotEmpty(token)) {
            // 从隐式参数中，获得 Token 值。
            Class<?> serviceType = invoker.getInterface();
            Map<String, String> attachments = inv.getAttachments();
            String remoteToken = attachments == null ? null : attachments.get(Constants.TOKEN_KEY);
            // 对比，若不一致，抛出 RpcException 异常
            if (!token.equals(remoteToken)) {
                throw new RpcException("Invalid token! Forbid invoke remote service " + serviceType + " method " + inv.getMethodName() + "() from consumer " + RpcContext.getContext().getRemoteHost() + " to provider " + RpcContext.getContext().getLocalHost());
            }
        }
        // 服务调用
        return invoker.invoke(inv);
    }

}
```

