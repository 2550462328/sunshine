裸es的9200端口任何人都可以访问，需要做访问限制。限制方案有：

1. 防火墙加白名单，只允许特定的ip访问。
2. nginx反向代理加basic auth
3. 给es安装xpack插件，限定用户用账号名-密码访问。
4. http_basic插件
5. search_guard插件



#### 1. 防火墙的设定方法：

CentOS默认防火墙是firewalld：

```
vi /etc/firewalld/zones/public.xml
```



开放除9200外所有端口，对9200只允许指定ip访问：

```
<?xml version="1.0" encoding="utf-8"?>
<zone>
  <short>Public</short>
  <description>For use in public areas. You do not trust the other computers on networks to not harm your computer. Only selected incoming connections are accepted.</description>
  <service name="ssh"/>
  <service name="dhcpv6-client"/>
  <port protocol="tcp" port="1-9199"/>
  <port protocol="tcp" port="9201-65535"/>
  <port protocol="udp" port="1-9199"/>
  <port protocol="udp" port="9201-65535"/>
  <rule family="ipv4">
    <source address="10.5.119.16"/>
    <port protocol="tcp" port="9200"/>
    <accept/>
  </rule>
</zone>
```



启动firewall并设置开机启动：

```
systemctl start firewalldsystemctl enable firewalld
```



#### 2. Nginx反向代理加basic auth

```
#es：
transport.bind_host: 172.31.197.24
network.bind_host: 127.0.0.1
#network.publish_host: 127.0.0.1
#network.host: 172.31.197.24
#
# Set a custom port for HTTP:
#
http.port: 9201

#nginx：
server {
    listen       9200;
    server_name  172.31.197.24;

    #charset koi8-r;
    #access_log  /var/log/nginx/host.access.log  main;
     
    location / {
        proxy_http_version 1.1;
        proxy_set_header   Connection          "";
        proxy_set_header   Host             $host;
        proxy_set_header   X-Real-IP        $remote_addr;
        proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
     
        auth_basic "login";
        auth_basic_user_file /etc/nginx/pass/es;
        autoindex on;
     
        proxy_pass http://localhost:9201/;
    }
}
```



账号密码由 https://www.sojson.com/htpasswd.html 生成的串放入/etc/nginx/pass/es即可



#### 3. 给es安装xpack并启用：

xpack license1个月过期后需要付费(证书可以免费续，但免费的证书没有安全功能)。

```
cd /opt/elasticsearch-5.6.3/bin
./elasticsearch-plugin install x-pack
vi ../conf/elasticsearch.yml
```



加上：

```
cluster.routing.allocation.disk.threshold_enabled: false
http.cors.allow-headers: Authorization,X-Requested-With,Content-Length,Content-Type
xpack.security.enabled: true
```



重启es即可。

xpack默认账户为elastic，默认密码为changeme。



访问esheader时，在esheader链接上加上权限请求参数，即可正常访问。如： http://172.31.197.24:9100/?auth_user=elastic&auth_password=changeme



#### 4. http_basic插件

只支持到1.5.1版本



#### 5. search_guard插件

安装后执行install_demo.sh， 启动后要执行sg_admin_demo.sh

java8以上会遇到 java.lang.NoClassDefFoundError: javax/xml/bind/DatatypeConverter 报错，需要找一个javax.xml.bind.jar放到es的lib目录下

启动后浏览器访问9200端口需要basic auth才能登录。但同时9300端口也需要证书才能登录。