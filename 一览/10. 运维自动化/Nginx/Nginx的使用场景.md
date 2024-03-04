#### 1. 反向代理

反向代理示意图如下：

![](https://pcc.huitogo.club/nginx9.png)



反向代理配置如下：

![](https://pcc.huitogo.club/nginx10.png)

注：proxy_pass http://mybackendserver/   后边这个斜线加和不加区别挺大的，**加的话不会拼接/backend  , 而不加的话会拼接 /backend。**



反向代理流程与原理

![](https://pcc.huitogo.club/nginx11.png)



#### 2. 负载均衡

负载均衡就是：避免高并发高流量时请求都聚集到某一个服务或者某几个服务上，而是让其均匀分配（或者能者多劳），从而减少高并发带来的系统压力，从而让服务更稳定。对于nginx来说，负载均衡就是从 upstream 模块定义的后端服务器列表中按照配置的负载策略选取一台服务器接受用户的请求。



示例配置

![](https://pcc.huitogo.club/nginx12.png)



nginx常用的负载策略

![](https://pcc.huitogo.club/nginx13.png)



#### 3. 动静分离

**为何要做动静分离?**

首先，我们常见的web系统中会有大量的静态资源文件比如掘金主页面刷新后的f12如下：

![](https://pcc.huitogo.club/nginx14.png)



可以看到有很多静态资源，如果将这些资源都搞到后端服务的话，将会提高后端服务的压力且占用带宽**增加了系统负载**（要知道，静态资源的访问频率其实蛮高的）所以为了避免该类问题我们可以把不常修改的静态资源文件放到nginx的静态资源目录中去，这样在访问静态资源时直接读取nginx服务器本地文件目录之后返回，这样就大大减少了后端服务的压力同时也加快了静态资源的访问速度。



何为静，何为动呢？：

1. 「静：」 将不常修改且访问频繁的静态文件，放到nginx本地静态目录（当然也可以搞个静态资源服务器专门存放所有静态文件）
2. 「动：」 将变动频繁/实时性较高的比如后端接口，实时转发到对应的后台服务



动静分离配置示例

![](https://pcc.huitogo.club/nginx15.png)



#### 4. 跨域

**为什么要跨域？**

产生跨域问题的主要原因就在于**同源策略**，为了**保证用户信息安全，防止恶意网站窃取数据**，同源策略是必须的，该政策由 Netscape 公司于1995年引入浏览器。目前，所有浏览器都实行这个政策。同源策略主要是指三点相同即：「**协议+域名+端口** 相同的两个请求」，则**可以被看做「是同源」的**，但如果「其中任意一点存在不同」，则代表是「两个不同源的请求」，**同源策略会限制不同源之间的资源交互从而减少数据安全问题。**



nginx解决跨域示例

![](https://pcc.huitogo.club/nginx16.png)



也就是将原本不同源的服务或资源放置在同一个协议+域名+端口下



#### 5. 缓存

nginx代理缓存可以在某些场景下有效的减少服务器压力，让请求快速响应，从而提升用户体验和服务性能.



nginx缓存配置参数

| 指令名称                 | 作用解释                                                     | 语法                                                         | 默认配置                               | 示例                                                         | 作用域                 |
| ------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | -------------------------------------- | ------------------------------------------------------------ | ---------------------- |
| proxy_cache              | 设置是否开启对后端响应的缓存。                               | proxy_cache zone \| off;                                     | proxy_cache off;                       | proxy_cache mycache; # 规定开启nginx缓存并且缓存名称为: mycache | http, server, location |
| proxy_cache_valid        | 配置什么状态码可以被缓存，以及缓存时长                       | proxy_cache_valid [code ...] time;                           | 没有默认值                             | proxy_cache_valid 200 304 2m; # 对于状态为200和304的缓存文件，缓存时间是2分钟 | http, server, location |
| proxy_cache_key          | 设置缓存文件的 key                                           | proxy_cache_key string;                                      | proxy_cache_keyproxy_host$request_uri; | proxy_cache_key "request_uri $cookie_user"; # 使用host +请求的uri以及cookie拼接成缓存key | http, server, location |
| proxy_cache_path         | 指定缓存存储的路径，文件名为cache key的md5值，然后多级目录的话，根据level参数来生成，key_zone参数用来指定在共享内存中缓存数据的名称和内存大小，比如keys_zone=mycache:100m，inactive用来指定缓存没有被访问后超时移除的时间，默认是10分钟，也可以自己指定比如inactive=2h ；max_size 用来指定缓存的最大值，超过这个值则会自动移除最近最少使用（lru淘汰算法）的缓存 这个指令对应的参数很多，具体见官网：proxy_cache_path | proxy_cache_path path [levels=levels] [use_temp_path=on\|off] keys_zone=name:size [inactive=time] [max_size=size] [min_free=size] [manager_files=number] [manager_sleep=time] [manager_threshold=time] [loader_files=number] [loader_sleep=time] [loader_threshold=time] [purger=on\|off] [purger_files=number] [purger_sleep=time] [purger_threshold=time]; | 无                                     | proxy_cache_path /data/nginx/cache levels=1:2 keys_zone=mycache:128m inactive=3d max_size=2g; # 设置缓存存放的目录为/data/nginx/cache，并设置缓存名称为mycache，大小为128m， 三天未被访问过的缓存将自动清除，磁盘中缓存的最大容量为2GB。 | http                   |
| proxy_cache_bypass       | 定义不从缓存中获取响应数据的条件。如果字符串参数中至少有一个值不为空且不等于" 0 "，则不会从缓存中获取响应: | proxy_cache_bypass string ...;                               | 没有默认值                             | proxy_cache_bypassarg_nocache$arg_comment;                   | http, server, location |
| proxy_cache_min_uses     | 指定某一个相同请求在几次请求之后才缓存响应内容               | proxy_cache_min_uses number;                                 | proxy_cache_min_uses 1;                | proxy_cache_min_uses 3; 规定某一个请求在第3次之后才走nginx缓存 | http, server, location |
| proxy_cache_use_stale    | 指定后端服务器在返回什么状态码的情况下可以使用过期的缓存     | proxy_cache_use_stale error timeout invalid_header http_500 http_502 http_503 ... \|off ; | proxy_cache_use_stale off;             | proxy_cache_use_stale error timeout http_500 http_502 http_503 http_504; # 规定服务在出现error timeout,以及502,503,504时可使用过期缓存 | http, server, location |
| proxy_cache_lock         | 默认不开启，开启后若出现并发重复请求，nginx只让一个请求去后端读数据，其他的排队并尝试从缓存中读取; | proxy_cache_lock on \|off;                                   | proxy_cache_lock off;                  | proxy_cache_lock on; # 开启缓存锁                            | http, server, location |
| proxy_cache_lock_timeout | 等待缓存锁(proxy_cache_lock)超时之后将直接请求后端，且结果不会被缓存 | proxy_cache_lock_timeout time;                               | proxy_cache_lock_timeout 5s;           | proxy_cache_lock_timeout 6s; # 等待缓存锁超时（6ms）之后将直接请求后端，结果不会被缓存。 | http, server, location |
| proxy_cache_methods      | 如果客户端请求方法在该指令中，则响应将被缓存。“GET”和“HEAD”方法总是被添加到列表中，尽管建议显式地指定它 | proxy_cache_methods GET\|HEAD \|POST ...;                    | proxy_cache_methods GET HEAD;          | proxy_cache_methods GET HEAD PUT POST; # 规定 可缓存的方法有 ：get head put post | http, server, location |



nginx缓存示例

```
http{
    ...
    # 指定缓存存放目录为/usr/local/nginx/test/nginx_cache_storage，并设置缓存名称为mycache，大小为64m， 1天未被访问过的缓存将自动清除，磁盘中缓存的最大容量为1gb
    proxy_cache_path /usr/local/nginx/test/nginx_cache_storage levels=1:2 keys_zone=mycache:64m inactive=1d max_size=1g;
    ...
    
    server{
        ...
        #  指定 username 参数中只要有字母 就不走nginx缓存  
        if ($arg_username ~ [a-z]) {
             set $cache_name "no cache";
        }
        
        location  /interface {
                   proxy_pass http://mybackendserver/;
                   # 使用名为 mycache 的缓存空间
                   proxy_cache mycache;
                   # 对于200 206 状态码的数据缓存2分钟
                   proxy_cache_valid 200 206 1m;
                   # 定义生成缓存键的规则（请求的url+参数作为缓存key）
                   proxy_cache_key $host$uri$is_args$args;
                   # 资源至少被重复访问2次后再加入缓存
                   proxy_cache_min_uses 3;
                   # 出现重复请求时，只让其中一个去后端读数据，其他的从缓存中读取
                   proxy_cache_lock on;
                   # 上面的锁 超时时间为4s，超过4s未获取数据，其他请求直接去后端
                   proxy_cache_lock_timeout 4s;
                   # 对于请求参数中有字母的 不走nginx缓存
                   proxy_no_cache $cache_name; # 判断该变量是否有值，如果有值则不进行缓存，没有值则进行缓存
                   # 在响应头中添加一个缓存是否命中的状态（便于调试）
                   add_header Cache-status $upstream_cache_status;    
        }
        ...
}
```



ps: 在上边配置文件中除了缓存相关的配置，我们还加了一个参数：

```
add_header Cache-status $upstream_cache_status;
```

这个参数可以方便从响应头看到是否命中了nginx缓存，方便我们观察，其不同的值有不同的含义。


upstream_cache_status的值集合如下：

- MISS：请求未命中缓存
- HIT：请求命中缓存。
- EXPIRED：请求命中缓存但缓存已过期。
- STALE：请求命中了陈旧缓存。
- REVALIDDATED：Nginx验证陈旧缓存依然有效。
- UPDATING：命中的缓存内容陈旧，但正在更新缓存。
- BYPASS：响应结果是从原始服务器获取的。



#### 6. 黑白名单

nginx黑白名单比较简单，allow后配置你的白名单，deny后配置你的黑名单，在实际使用中，我们一般都是建个黑名单和白名单的文件然后再nginx.copnf中incluld一下，这样保持主配置文件整洁，也好管理。



官方示例如下：

![](https://pcc.huitogo.club/nginx17.png)

可以看到ip 可以是ipv4 也可以是ipv6 也可以按照网段来配置，当然ip黑白配置可以在 http，server，location和limit_except这几个域都可以区别只是作用粒度大小问题。当然nginx建议我们使用 ngx_http_geo_module这个库，ngx_http_geo_module库支持 按地区、国家进行屏蔽，并且提供了IP库，当需要配置的名单比较多或者根据地区国家屏蔽时这个库可以帮上大忙。



#### 7. 限流

Nginx主要有两种限流方式：按并发连接数限流(ngx_http_limit_conn_module)、按请求速率限流(ngx_http_limit_req_module 使用的令牌桶算法)。



**1）对请求速率限流**

```
http{
    ...
    # 对请求速率限流
    limit_req_zone $binary_remote_addr zone=myRateLimit:10m rate=5r/s;
    
    server{
        location /interface{
            ...
            limit_req zone=myRateLimit burst=5  nodelay;
            limit_req_status 520;
            limit_req_log_level info;
        }
    }
}
```

- 「**$binary_remote_addr**」：表示基于 remote_addr(客户端IP) 来做限流「**zone=myRateLimit:10m**」：表示使用myRateLimit来作为内存区域（存储访问信息）的名字，大小为10M，1M能存储16000 IP地址的访问信息，10M可以存储16W IP地址访问信息。
- 「**rate=5r/s**」：表示相同ip每秒最多请求5次，nginx是精确到毫秒的，也就是说此配置代表每200毫秒处理一个请求，这意味着自上一个请求处理完后，若后续200毫秒内又有请求到达，将拒绝处理该请求（如果没配burst的话）。
- 「**burst=5**」：(英文 爆发 的意思)，意思是设置一个大小为5的缓冲队列，若同时有6个请求到达，Nginx 会处理第一个请求，剩余5个请求将放入队列，然后每隔200ms从队列中获取一个请求进行处理。若请求数大于6，将拒绝处理多余的请求，直接返回503。
- 「**nodelay**」：针对的是 burst 参数，burst=5 nodelay 这个配置表示被放到缓冲队列的这5个请求会立马处理，不再是每隔200ms取一个了。但是值得注意的是，即使这5个突发请求立马处理并结束，后续来了请求也不一定不会立马处理，因为虽然请求被处理了但是请求所占的坑并不会被立即释放，而是只能按 200ms 一个来释放，释放一个后 才将等待的请求 入队一个。
- 「另外两个：」 limit_req_status=520表示当被限流后，nginx的返回码，limit_req_log_level info代表日志级别



注：

- 如果不开启nodelay且开启了burst这个配置，那么将会严重影响用户体验（你想想假设burst队列长度为100的话每100ms处理一个,那队列最后那个请求得等10000ms=10s后才能被处理，那不超时才怪呢此时burst已经意义不大了）所以**一般情况下 建议burst和nodelay结合使用**，从而尽可能达到速率稳定，但突然流量也能正常处理的效果。
- nodelay参数本质上并没有提高访问速率，而仅仅是让处于burst队列的请求 ”被快速处理“ 罢了。



**2）对连接数量数量**

```
http{
    # 针对ip  对请求连接数限流
    ...
    limit_conn_zone $binary_remote_addr zone=myConnLimit:10m; 
    ...
    
    server{
       ...
       limit_conn myConnLimit 12;
    }
}    
```



针对连接数量的限流和速率不一样，即使你速率是1ms一次，只要你连接数量不超过设置的，那么也访问成功。如果连接数超过设置的值将会请求失败。



#### 8. 配置SSL

配置说明

```
    server {
        #SSL 默认访问端口号为 443
        listen 443 ssl; 
        #填写绑定证书的域名 
        server_name www.hzznb-xzll.xyz hzznb-xzll.xyz; 
        #请填写证书文件的相对路径或绝对路径
        ssl_certificate /usr/local/nginx/certificate/hzznb-xzll.xyz_bundle.crt; 
        #请填写私钥文件的相对路径或绝对路径
        ssl_certificate_key /usr/local/nginx/certificate/hzznb-xzll.xyz.key; 
        #停止通信时，加密会话的有效期，在该时间段内不需要重新交换密钥
        ssl_session_timeout 5m;
        #服务器支持的TLS版本
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3; 
        #请按照以下套件配置，配置加密套件，写法遵循 openssl 标准。
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE; 
        #开启由服务器决定采用的密码套件
        ssl_prefer_server_ciphers on;
    }    
```



http跳转https

```
server_name www.hzznb-xzll.xyz hzznb-xzll.xyz;
# 重定向到目标地址
return 301 https://$server_name$request_uri;
```



#### 9. 压缩

压缩功能比较实用尤其是处理一些大文件时，而gzip 是规定的三种标准 HTTP 压缩格式之一。目前绝大多数的网站都在使用 gzip 传输 HTML 、CSS 、 JavaScript 等资源文件。需要知道的是，并不是每个浏览器都支持 gzip 压缩，如何知道客户端（浏览器）是否支持 压缩 呢？ 可以通过观察 某请求头中的 Accept-Encoding 来观察是否支持压缩，另外只有客户端支持也不顶事，服务端得返回gzip格式的文件呀，那么这件事nginx可以帮我们做，我们可以通过 Nginx 的配置来让服务端支持 gzip。服务端返回压缩文件后浏览器进行解压缩从而展示正常内容。



压缩配置示例：

```
http {
    # 开启/关闭 压缩机制
    gzip on;
    # 根据文件类型选择 是否开启压缩机制
    gzip_types text/plain application/javascript text/css application/xml text/javascript image/jpeg image/jpg image/gif image/png  application/json;
    # 设置压缩级别，一共9个级别  1-9   ，越高资源消耗越大 越耗时，但压缩效果越好，
    gzip_comp_level 9;
    # 设置是否携带Vary:Accept-Encoding 的响应头
    gzip_vary on;
    # 处理压缩请求的 缓冲区数量和大小
    gzip_buffers 32 64k;
    # 对于不支持压缩功能的客户端请求 不开启压缩机制
    gzip_disable "MSIE [1-6]\."; # 比如低版本的IE浏览器不支持压缩
    # 设置压缩功能所支持的HTTP最低版本
    gzip_http_version 1.1;
    # 设置触发压缩的最小阈值
    gzip_min_length 2k;
    # off/any/expired/no-cache/no-store/private/no_last_modified/no_etag/auth 根据不同配置对后端服务器的响应结果进行压缩
    gzip_proxied any;
} 
```



注：压缩后虽然体积变小了，但是响应的时间会变长，因为压缩/解压也需要时间呀！压缩功能似乎有点：「**用时间换空间的感觉**！」，当然压缩级别可以调的，你可以选择较低级别的压缩，这样既能实现压缩功能使得数据包体积降下来，同时压缩时间也会缩短是比较折中的一种方案。



**10. 重写**

rewrite指令是通过正则表达式来改变URI。可以同时存在一个或多个指令。需要按照顺序依次对URL进行匹配和处理，常用于重定向功能。

**语法：rewrite 正则表达式 要替换的内容 [flag];**



其中flag有如下几个值：

- 「last:」  本条规则匹配完成后，继续向下匹配新的location URI 规则。
- 「break:」  本条规则匹配完成即终止，不再匹配后面的任何规则。
- 「redirect:」  返回302临时重定向，浏览器地址会显示跳转新的URL地址。
- 「permanent:」  返回301永久重定向。浏览器地址会显示跳转新的URL地址。



rewrite示例

```
  server {
      listen 80 default;
      charset utf-8;
      server_name www.hzznb-xzll.xyz hzznb-xzll.xyz;

      # 临时（redirect）重定向配置
      location /temp_redir {
          rewrite ^/(.*) https://www.baidu.com redirect;
      }
      # 永久重定向（permanent）配置
      location /forever_redir {

          rewrite ^/(.*) https://www.baidu.com permanent;
      }

      # rewrite last配置
      location /1 {
        rewrite /1/(.*) /2/$1 last;
      }
      location /2 {
        rewrite /2/(.*) /3/$1 last;
      }
      location /3 {
        alias  '/usr/local/nginx/test/static/';
        index location_last_test.html;
      }
    }
```



示例测试结果：

- 「last配置:」 可以看到我们定义 访问 http://hzznb-xzll.xyz/1/ 的请求被替换为 http://hzznb-xzll.xyz/2/ 之后再被替换为 http://hzznb-xzll.xyz/3/  最后找到/usr/local/nginx/test/static/location_last_test.html 这个文件并返回。
- 「redirect配置：」 当访问 http://hzznb-xzll.xyz/temp_redir/ 这个请求会临时（302）重定向到百度页面
- 「permanent配置：」 当访问 http://hzznb-xzll.xyz/forever_redir/ 这个请求会永久（301）重定向到百度页面



#### 11. auto_index

一般用于快速搭建静态资源网站，比如我要给自己搞个书籍网站里边放些好书，希望有个站点可以直接翻阅这个书本目录



配置示例

```
location /book/ {
          root /usr/local/nginx/test;
          autoindex on; # 打开 autoindex，，可选参数有 on | off
          autoindex_format html; # 以html的方式进行格式化，可选参数有 html | json | xml
          autoindex_exact_size on; # 修改为off，会以KB、MB、GB显示文件大小，默认为on以bytes显示出⽂件的确切⼤⼩
          autoindex_localtime off; # 显示⽂件时间 GMT格式
      }
```



示例效果

![](https://pcc.huitogo.club/nginx18.png)