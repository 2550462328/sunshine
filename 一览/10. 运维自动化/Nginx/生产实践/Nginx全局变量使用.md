Nginx支持if语法，if语法可以根据Nginx全局变量值进行判定，比如根据url  参数 ip 域名等等做比对或者判断（一般都使用正则的方式），例如：

```
     # 指定 username 参数中只要有字母 就不走nginx缓存
     if ($arg_username ~ [a-z]) {
         set $cache_name "no cache";
     }
```



全局变量有：

| 变量                 | 解释                                                         |
| -------------------- | ------------------------------------------------------------ |
| $time_local          | 本地时间                                                     |
| $time_iso8601        | ISO 8601 时间格式                                            |
| $arg_name            | 请求中的的参数名，即“?”后面的arg_name=arg_value形式的arg_name |
| $args                | 与$query_string相同 等于URL当中的参数(GET请求时)，如a=1&b=2  |
| $document_uri        | 与相同这个变量指当前的请求，不包括任何参数见args)            |
| $request_uri         | 包含请求参数的原始URI，不包含主机名，如：/aaa/bbb.html?a=1&b=2 |
| $is_args             | 如果URL包含参数则为？，否则为空字符串                        |
| $query_string        | 与$args相同 等于URL当中的参数(GET请求时)，如a=1&b=2          |
| $uri                 | 当前请求的URI,不包含任何参数                                 |
| $remote_addr         | 获取客户端ip                                                 |
| $binary_remote_addr  | 客户端ip（二进制)                                            |
| $remote_port         | 客户端port                                                   |
| $remote_user         | 用于基本验证的用户名。                                       |
| $host                | 请求主机头字段，否则为服务器名称，如:https://www.hzznb-xzll.xyz |
| $proxy_host          | proxy_pass 指令设置的后端服务器的域名（或者IP地址）          |
| $proxy_port          | proxy_pass 指令设置的后端服务器的监听端口。                  |
| $http_host           | 是和server_port 两个变量的结合                               |
| $request             | 用户请求信息，如：GET ?a=1&b=2 HTTP/1.1                      |
| $request_time        | 请求所用时间，单位毫秒                                       |
| $request_method      | 请求的方法 比如 get post put delete update 等                |
| $request_filename    | 当前请求的文件的路径名，由root或alias和URI request组合而成,如：/aaa/bbb.html |
| $status              | 请求的响应状态码,如：200                                     |
| $body_bytes_sent     | 响应时送出的body字节数数量。即使连接中断，这个数据也是精确的,如：40，传输给客户端的字节数，响应头不计算在内；这个变量和Apache的mod_log_config模块中的“%B”参数保持兼容 |
| $content_length      | 等于请求行的“Content_Length”的值                             |
| $content_type        | 等于请求行的“Content_Type”的值                               |
| $http_referer        | 引用地址                                                     |
| $http_user_agent     | 客户端agent信息,如：Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/119.0.0.0 Safari/537.36 这个可以用来区分手机端还是pc端 |
| $document_root       | 针对当前请求的根路径设置值                                   |
| $hostname            | 主机名称                                                     |
| $http_cookie         | 客户端cookie信息                                             |
| $cookie_COOKIE       | cookie COOKIE变量的值                                        |
| $limit_rate          | 这个变量可以限制连接速率，0表示不限速                        |
| $request_body        | 记录POST过来的数据信息                                       |
| $request_body_file   | 客户端请求主体信息的临时文件名                               |
| $scheme              | HTTP方法（如http，https）                                    |
| $request_completion  | 如果请求结束，设置为OK. 当请求未结束或如果该请求不是请求链串的最后一个时，为空(Empty)，如：OK |
| $server_protocol     | 请求使用的协议，通常是HTTP/1.0或HTTP/1.1，如：HTTP/1.1       |
| $server_addr         | 服务器IP地址，在完成一次系统调用后可以确定这个值             |
| $server_name         | 响应请求的服务器名称                                         |
| $server_port         | 请求到达服务器的端口号，如：80                               |
| $connection          | 连接序列号                                                   |
| $connection_requests | 当前通过连接发出的请求数                                     |
| $nginx_version       | nginx版本                                                    |
| $pid                 | 工作进程的PID                                                |
| $pipe                | 如果请求来自管道通信，值为“p”，否则为“.”                     |
| $proxy_protocol_addr | 获取代理访问服务器的客户端地址，如果是直接访问，该值为空字符串 |
| $realpath_root       | 对应于当前请求的根目录或别名值的绝对路径名，所有符号连接都解析为真实路径。 |