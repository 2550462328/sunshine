#### 1. nginx负载

![img](http://pcc.huitogo.club/7bce06a8f86f8c8dbe23e2f3912bd41c)

nginx负载分为 轮询、ip哈希、加权轮询

nginx会自动剔除宕机的服务器



配置如下：

- **轮询**

```
1.  upstream backserver {   

2.  server 192.168.1.14;   

3.  server 192.168.1.15;   

4.  }   
```



- **ip哈希**

```
1.  upstream backserver {   

2.  ip_hash;   

3.  server 192.168.0.14;   

4.  server 192.168.0.15;   

5.  }   
```



- **加权轮询**

```
1.  upstream backserver {   

2.  server 192.168.1.14 weight=1;   

3.  server 192.168.1.15 weight=2;   

4.  }   
```



**路由重试**

配置重试次数和重试最大时间如下：

```
1.  upstream backserver {   

2.  server 192.168.1.14  weight=1  max_fails=2 fail_timeout=30s;   

3.  server 192.168.1.15  weight=2  max_fails=2 fail_timeout=30s;  

4.  }   
```



**热机（备机）**

```
1.  upstream backserver {   

2.  server 192.168.1.14  weight=1  max_fails=2 fail_timeout=30s;   

3.  server 192.168.1.15  weight=2  max_fails=2 fail_timeout=30s;  

4.  server 192.168.1.16 backup;  

5.  }  
```

当所有的非备机（non-backup）都宕机（down）或者繁忙（busy）的时候，就会使用由backup 标注的备机。必须要注意的是，backup 不能和 ip_hash 关键字一起使用。



#### 2. nginx限流

nginx.conf中限流配置如下

```
1.  #统一在http域中进行配置  

2.  #限制请求  

3.  limit_req_zone $binary_remote_addr $uri zone=api_read:20m rate=50r/s;  

4.  #按ip配置一个连接 zone  

5.  limit_conn_zone $binary_remote_addr zone=perip_conn:10m;  

6.  #按server配置一个连接 zone  

7.  limit_conn_zone $server_name zone=perserver_conn:100m;  

8.  server {  

9.          listen       80;  

10.         server_name  report.52itstyle.com;  

11.         index login.jsp;  

12.         location / {  

13.               #请求限流排队通过 burst默认是0  

14.               limit_req zone=api_read burst=5;  

15.               #连接数限制,每个IP并发请求为2  

16.               limit_conn perip_conn 2;  

17.               #服务所限制的连接数(即限制了该server并发连接数量)  

18.               limit_conn perserver_conn 1000;  

19.               #连接限速  

20.               limit_rate 100k;  

21.               proxy_pass      http://report;  

22.         }  

23. }  
```



其中

- limit_conn_zone：是针对每个IP定义一个存储session状态的容器。这个示例中定义了一个100m的容器，按照32bytes/session，可以处理3200000个session。

- limit_rate 300k：对每个连接限速300k.

  注意，这里是对连接限速，而不是对IP限速。如果一个IP允许两个并发连接，那么这个IP就是限速limit_rate×2。

- burst=5：就是在连接数满的情况排队等待的最大数量，如果排队等待的数量达到最大数量，后续连接请求直接驳回，一般配合超时使用，就是排队也有个限时的。



请求失败的连接可以跳转到自定义的错误页面，nginx中配置如下

```
1.  error_page   500 502 503 504  /50x.html;  

2.  location = /50x.html {  

3.      root   html;#自定义50X错误  

4.  }  
```



温馨提示：nginx的限流功能需要安装以下模块

- ngx_http_limit_conn_module (static)
- ngx_http_limit_req_module (static)



#### 3. nginx缓存配置

基本配置如下，主要是开启缓存

```
1.  proxy_cache_path /path/to/cache levels=1:2 keys_zone=mycache:10m max_size=10g inactive=60m use_temp_path=off;  

2.  server {  

3.  　　# ...  

4.  　　location / {  

5.  　　　　proxy_cache mycache;  

6.  　　　　proxy_pass http://my_upstream;  

7.  　　}  

8.  }  
```



其中

- **/path/to/cache**

本地路径，用来设置Nginx缓存资源的存放地址

- **levels**

默认所有缓存文件都放在同一个/path/to/cache下，但是会影响缓存的性能，因此通常会在/path/to/cache下面建立子目录用来分别存放不同的文件。假设levels=1:2，Nginx为将要缓存的资源生成的key为f4cd0fbc769e94925ec5540b6a4136d0，那么key的最后一位0，以及倒数第2-3位6d作为两级的子目录，也就是该资源最终会被缓存到/path/to/cache/0/6d目录中

- **key_zone**

在共享内存中设置一块存储区域来存放缓存的key和metadata（类似使用次数），这样nginx可以快速判断一个request是否命中或者未命中缓存，1m可以存储8000个key，10m可以存储80000个key

- **max_size**

最大cache空间，如果不指定，会使用掉所有disk space，当达到配额后，会删除最少使用的cache文件

- **inactive**

未被访问文件在缓存中保留时间，本配置中如果60分钟未被访问则不论状态是否为expired，缓存控制程序会删掉文件。inactive默认是10分钟。需要注意的是，inactive和expired配置项的含义是不同的，expired只是缓存过期，但不会被删除，inactive是删除指定时间内未被访问的缓存文件

- **use_temp_path**

如果为off，则nginx会将缓存文件直接写入指定的cache文件中，而不是使用temp_path存储，official建议为off，避免文件在不同文件系统中不必要的拷贝

- **proxy_cache**

启用proxy cache，并指定key_zone。另外，如果proxy_cache off表示关闭掉缓存。

- **proxy_pass**

设置被代理server的协议和地址，协议可以为http或https，地址可以为域名或IP，需要注意的是如果这个地址后面带/，如http://127.0.0.1/，然后你调用地址，它会直接带上请求路径，也就是http://127.0.0.1/index.html，但是没有带上/，如http://127.0.0.1，那么当你调用地址，它还会加上localtion中的请求路径，也就是http://127.0.0.1/myserver/index.html