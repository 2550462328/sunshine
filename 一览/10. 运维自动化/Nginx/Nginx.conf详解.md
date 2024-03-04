nginx.conf文件是由一个一个的指令块组成的，nginx用{}标识一个指令块，精简后的结构如下：

```
全局模块
event模块
http模块
    upstream模块
    
    server模块
        localtion块
        localtion块
        ....
    server模块
        localtion块
        localtion块
        ...
    ....    
```

1. 「全局模块：」 配置影响nginx全局的指令，比如运行nginx的用户名，nginx进程pid存放路径，日志存放路径，配置文件引入，worker进程数等。

2. 「events块：」 配置影响nginx服务器或与用户的网络连接。比如每个进程的最大连接数，选取哪种事件驱动模型（select/poll epoll或者是其他等等nginx支持的）来处理连接请求，是否允许同时接受多个网路连接，开启多个网络连接序列化等。

3. 「http块：」 可以嵌套多个server，配置代理，缓存，日志格式定义等绝大多数功能和第三方模块的配置。如文件引入，mime-type定义，日志自定义，是否使用sendfile传输文件，连接超时时间，单连接请求数等。

4. 「server块：」 配置虚拟主机的相关参数比如域名端口等等，一个http中可以有多个server。

5. 「location块：」 配置url路由规则。

6. 「upstream块：」 配置上游服务器的地址以及负载均衡策略和重试策略等等。



示例nginx.conf

```
#user  nobody; # 指定Nginx Worker进程运行用户以及用户组，默认由nobody账号运行
worker_processes  1;  # 指定工作进程的个数，默认是1个。具体可以根据服务器cpu数量进行设置， 比如cpu有4个，可以设置为4。如果不知道cpu的数量，可以设置为auto。 nginx会自动判断服务器的cpu个数，并设置相应的进程数
#error_log  logs/error.log;  # 用来定义全局错误日志文件输出路径，这个设置也可以放入http块，server块，日志输出级别有debug、info、notice、warn、error、crit可供选择，其中，debug输出日志最为最详细，而crit输出日志最少。
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info; # 指定error日志位置和日志级别
#pid        logs/nginx.pid;  # 用来指定进程pid的存储文件位置

events {
    accept_mutex on;   # 设置网路连接序列化，防止惊群现象发生，默认为on
    
    # Nginx支持的工作模式有select、poll、kqueue、epoll、rtsig和/dev/poll，其中select和poll都是标准的工作模式，kqueue和epoll是高效的工作模式，不同的是epoll用在Linux平台上，而kqueue用在BSD系统中，对于Linux系统，epoll工作模式是首选
    use epoll;
    
    # 用于定义Nginx每个工作进程的最大连接数，默认是1024。最大客户端连接数由worker_processes和worker_connections决定，即Max_client=worker_processes*worker_connections在作为反向代理时，max_clients变为：max_clients = worker_processes *worker_connections/4。进程的最大连接数受Linux系统进程的最大打开文件数限制，在执行操作系统命令“ulimit -n 65536”后worker_connections的设置才能生效
    worker_connections  1024; 
}
# 对HTTP服务器相关属性的配置如下
http {
    include       mime.types; # 引入文件类型映射文件 
    default_type  application/octet-stream; # 如果没有找到指定的文件类型映射 使用默认配置 
    # 设置日志打印格式
    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';
    # 
    #access_log  logs/access.log  main; # 设置日志输出路径以及 日志级别
    sendfile        on; # 开启零拷贝 省去了内核到用户态的两次copy故在文件传输时性能会有很大提升
    #tcp_nopush     on; # 数据包会累计到一定大小之后才会发送，减小了额外开销，提高网络效率
    keepalive_timeout  65; # 设置nginx服务器与客户端会话的超时时间。超过这个时间之后服务器会关闭该连接，客户端再次发起请求，则需要再次进行三次握手。
    #gzip  on; # 开启压缩功能，减少文件传输大小，节省带宽。
    sendfile_max_chunk 100k; #每个进程每次调用传输数量不能大于设定的值，默认为0，即不设上限。
    
    # 配置你的上游服务（即被nginx代理的后端服务）的ip和端口/域名
    upstream backend_server { 
        server 172.30.128.65:8080;
        server 172.30.128.65:8081 backup; #备机
    }

    server {
        listen       80; #nginx服务器监听的端口
        server_name  localhost; #监听的地址 nginx服务器域名/ip 多个使用英文逗号分割
        #access_log  logs/host.access.log  main; # 设置日志输出路径以及 级别，会覆盖http指令块的access_log配置
        
        # localtion用于定义请求匹配规则。 以下是实际使用中常见的3中配置（即分为：首页，静态，动态三种）
       
        # 第一种：直接匹配网站根目录，通过域名访问网站首页比较频繁，使用这个会加速处理，一般这个规则配成网站首页，假设此时我们的网站首页文件就是： usr/local/nginx/html/index.html
        location = / {  
            root   html; # 静态资源文件的根目录 比如我的是 /usr/local/nginx/html/
            index  index.html index.htm; # 静态资源文件名称 比如：网站首页html文件
        }
        # 第二种：静态资源匹配（静态文件修改少访问频繁，可以直接放到nginx或者统一放到文件服务器，减少后端服务的压力），假设把静态文件我们这里放到了 usr/local/nginx/webroot/static/目录下
        location ^~ /static/ {
            alias /webroot/static/; 访问 ip:80/static/xxx.jpg后，将会去获取/url/local/nginx/webroot/static/xxx.jpg 文件并响应
        }
        # 第二种的另外一种方式：拦截所有 后缀名是gif,jpg,jpeg,png,css.js,ico这些 类静态的的请求，让他们都去直接访问静态文件目录即可
        location ~* \.(gif|jpg|jpeg|png|css|js|ico)$ {
            root /webroot/static/;
        }
        # 第三种：用来拦截非首页、非静态资源的动态数据请求，并转发到后端应用服务器 
        location / {
            proxy_pass http://backend_server; #请求转向 upstream是backend_server 指令块所定义的服务器列表
            deny 192.168.3.29; #拒绝的ip （黑名单）
            allow 192.168.5.10; #允许的ip（白名单）
        }
        
        # 定义错误返回的页面，凡是状态码是 500 502 503 504 总之50开头的都会返回这个 根目录下html文件夹下的50x.html文件内容
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
        
    }
    # 其余的server配置 ,如果有需要的话
    #server {
        ......
    #    location / {
               ....
    #    }
    #}
    
    # include /etc/nginx/conf.d/*.conf;  # 一般我们实际使用中有很多配置，通常的做法并不是将其直接写到nginx.conf文件，
    # 而是写到新文件 然后使用include指令 将其引入到nginx.conf即可，这样使得主配置nginx.conf文件更加清晰。
    
}
```



注：有些指令是可以在不同作用域使用的，如果在多个作用域都有相同指令的使用，那么nginx将会遵循就近原则或者我愿称之为 「**最近配置优先**」。 eg: 你在 http配了日志级别，也在某个server中配了日志级别，那么这个server将使用他自己配置的已不使用外层的http日志配置。



#### 1. localtion 路由匹配

**什么是location**? : nginx根据用户请求的URI来匹配对应的location模块，匹配到哪个location，请求将被哪个location块中的配置项所处理。



**常见匹配规则**如下:

![](https://pcc.huitogo.club/nginx3.png)



**1）前缀匹配（无修饰符）**

测试配置

![](https://pcc.huitogo.club/nginx4.png)



测试结果

```
curl http://www.locatest.com/prefixmatch     ✅ 301
curl http://www.locatest.com/prefixmatch?    ✅ 301
curl http://www.locatest.com/PREFIXMATCH     ❌ 404
curl http://www.locatest.com/prefixmatch/    ✅ 200
curl http://www.locatest.com/prefixmatchmmm  ❌ 404
curl http://www.locatest.com/prefixmatch/mmm ❌ 404
curl http://www.locatest.com/aaa/prefixmatch/❌ 404
```

可以看到 域名/prefixmatch 和域名/prefixmatch? 返回了301 ，原因在于prefixmatch映射的 /etc/nginx/locatest/ **是个目录**，而不是个文件所以nginx**提示我们301**，这个我们不用管没关系，总之我们知道：域名/prefixmatch，域名/prefixmatch? 和域名/prefixmatch/ 这三个url通过我们配置的 「无修饰符前缀匹配规则」 都能匹配上就行了。



**2）精确匹配（ = ）**

测试配置

![](https://pcc.huitogo.club/nginx5.png)



测试结果

```
http://www.locatest.com/exactmatch      ✅ 200
http://www.locatest.com/exactmatch？    ✅ 200
http://www.locatest.com/exactmatch/     ❌ 404
http://www.locatest.com/exactmatchmmmm  ❌ 404
http://www.locatest.com/EXACTMATCH      ❌ 404
```

可以看出来精确匹配就是精确匹配，差一个字也不行！



**3）前缀匹配（ ^~ ）**

测试配置

![](https://pcc.huitogo.club/nginx6.png)



测试结果

```
curl http://www.locatest.com/exactprefixmatch     ✅ 200
curl http://www.locatest.com/exactprefixmatch/    ✅ 200
curl http://www.locatest.com/exactprefixmatch?    ✅ 200
curl http://www.locatest.com/exactprefixmatchmmm  ✅ 200
curl http://www.locatest.com/exactprefixmatch/mmm ✅ 200
curl http://www.locatest.com/aaa/exactprefixmatch ❌ 404
curl http://www.locatest.com/EXACTPREFIXMATCH     ❌ 404
```

可以看到带修饰符(^~)的前缀匹配 像：域名/exactprefixmatchmmm 和域名/exactprefixmatch/mmm 是可以匹配上的，其他的和不带修饰符的前缀匹配似乎都差不多。



**4）正则匹配（~ 区分大小写）**

测试配置

![](https://pcc.huitogo.club/nginx7.png)



测试结果

```
curl http://www.locatest.com/regexmatch      ✅ 200
curl http://www.locatest.com/regexmatch/     ❌ 404
curl http://www.locatest.com/regexmatch?     ✅ 200
curl http://www.locatest.com/regexmatchmmm   ❌ 404
curl http://www.locatest.com/regexmatch/mmm  ❌ 404
curl http://www.locatest.com/REGEXMATCH      ❌ 404
curl http://www.locatest.com/aaa/regexmatch  ❌ 404
curl http://www.locatest.com/bbbregexmatch   ❌ 404
```

可以看到~修饰的正则是**区分大小写**的。



**5）正则匹配（~\* 不区分大小写）**

测试配置

![](https://pcc.huitogo.club/nginx8.png)



测试结果

```
curl http://www.locatest.com/regexmatch      ✅ 200
curl http://www.locatest.com/regexmatch/     ❌ 404
curl http://www.locatest.com/regexmatch?     ✅ 200
curl http://www.locatest.com/regexmatchmmm   ❌ 404
curl http://www.locatest.com/regexmatch/mmm  ❌ 404
curl http://www.locatest.com/REGEXMATCH      ✅ 200
curl http://www.locatest.com/aaa/regexmatch  ❌ 404
curl http://www.locatest.com/bbbregexmatch   ❌ 404
```

可以看到这次 curl http://www.locatest.com/REGEXMATCH 是可以匹配上的，说明 ~* 确实是不区分大小写的。



**6）通用匹配（ / ）**

通用匹配使用一个 / 表示，可以匹配所有请求，一般nginx配置文件最后都会有一个通用匹配规则，当其他匹配规则均失效时，请求会被路由给通用匹配规则处理，如果没有配置通用匹配，并且其他所有匹配规则均失效时，nginx会返回404错误。



**location 匹配优先级：**

1. 优先走精确匹配，精确匹配命中时，直接走对应的location，停止之后的匹配动作。
2. 无修饰符类型的前缀匹配和 ^~ 类型的前缀匹配命中时，收集命中的匹配，对比出最长的那一条并存起来(最长指的是与请求url匹配度最高的那个location)。
3. 如果步骤2中最长的那一条匹配是^~类型的前缀匹配，直接走此条匹配对应的location并停止后续匹配动作；如果步骤2最长的那一条匹配不是^~类型的前缀匹配（也就是无修饰符的前缀匹配），则继续往下匹配
4. 按location的声明顺序，执行正则匹配，当找到第一个命中的正则location时，停止后续匹配。
5. 都没匹配到，走通用匹配（ / ）（如果有配置的话），如果没配置通用匹配的话，上边也都没匹配上，到这里就是404了。



简单理解下就是：**= >  ^~  > 正则 > 无修饰符的前缀匹配 > /**



#### 2.  root&alias

root和alias这俩货一般都是用于指**定静态资源目录**，区别在于**root会拼接匹配location路径，而alia不会**



例如以下配置

```
location /static/ {
    root /usr/local/nginx/test;
}
```

root指令会 将 /static/ 拼接到 /usr/local/nginx/test 后边 ，即完整目录路径为： /usr/local/nginx/test/static/

如果使用alia则完整录路径为： /usr/local/nginx/test/



注：使用alia 的目录后面一定要加/，这个可以联想到proxy_pass中的跳转目录也有加/和不加/的区别，总结就是

- 路径加/的说明指向的是目录，则不需要拼接匹配localtion路径，案例中就是static
- 路径不加/的说明不确定是不是一个目录，在proxy_pass中会拼接上location路径



#### 3. 重试策略

关于重试策略我们这里也说一下，重试是在发生错误时的一种不可缺少的手段，这样当某一个或者某几个服务宕机时（因为我们现在大多都是多实例部署），如果有正常服务，那么将请求 重试到正常服务的机器上去。



重试示例

```
    upstream mybackendserver {
        # 60秒内 如果请求8081端口这个应用失败
        # 3次，则认为该应用宕机 时间到后再有请求进来继续尝试连接宕机应用且仅尝试 1 次，如果还是失败，
        # 则继续等待 60 秒...以此循环，直到恢复
        server 172.30.128.64:8081 fail_timeout=60s max_fails=3; 
        
        server 172.30.128.64:8082;
        server 172.30.128.64:8083;
    }
```



当然我们可以选择在哪些返回状态下才允许重试

```
 # 指定哪些错误状态才执行 重试 比如下边的 error 超时，500,502,503 504
proxy_next_upstream error timeout http_500 http_502 http_503 http_504;
```



#### 4. backup

Nginx 支持设置备用节点，当所有线上节点都异常时会启用备用节点，同时备用节点也会影响到失败重试的逻辑。



我们可以通过 backup 指令来定义备用服务器，backup有如下特征：

1. 正常情况下，请求不会转到到 backup 服务器，包括失败重试的场景
2. 当所有正常节点全部不可用时，backup 服务器生效，开始处理请求
3. 一旦有正常节点恢复，就使用已经恢复的正常节点
4. backup 服务器生效期间，不会存在所有正常节点一次性恢复的逻辑
5. 如果全部 backup 服务器也异常，则会将所有节点一次性恢复，加入存活列表
6. 如果全部节点（包括 backup）都异常了，则 Nginx 返回 502 错误