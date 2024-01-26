#### 1. 容器访问端口映射

```
# 本机任意端口 映射到 容器的80端口

docker run -d -P nginx:alpine

# 本机80端口 映射到 容器的80端口

docker run -d -p 80:80 nginx:alpine

# 127.0.0.1的80端口 映射到 容器的80端口

docker run -d -p 127.0.0.1:80:80 nginx:alpine

# 127.0.0.1的任意端口 映射到 容器的80端口

docker run -d -p 127.0.0.1::80 nginx:alpine

# 标识udp端口

docker run -d -p 127.0.0.1:80:80/udp nginx:alpine

 

# 查看容器80端口映射情况

docker port fa 80

 

# 也可以声明多个端口映射

docker run -d \

  -p 80:80 \

  -p 443:443 \

  nginx:alpine
```



#### 2. 外部网络访问

容器要想访问外部网络，需要本地系统的转发支持。在Linux 系统中，检查转发是否打开。

```
sysctl net.ipv4.ip_forward
```

如果为 0，说明没有开启转发，则需要手动打开。

```
sysctl -w net.ipv4.ip_forward=1
```

如果在启动 Docker 服务的时候设定 --ip-forward=true, Docker 就会自动设定系统的 ip_forward 参数为 1。



#### 3. 容器通信

容器之间相互访问，需要两方面的支持。

- 容器的网络拓扑是否已经互联。默认情况下，所有容器都会被连接到 docker0 网桥上。
- 本地系统的防火墙软件 -- iptables 是否允许通过。

```
# 手动创建docker网络

docker network create -d bridge my-net

# 容器busybox1加入网络

docker run -it --rm --name busybox1 --network my-net busybox sh

# 容器busybox2加入网络

docker run -it --rm --name busybox2 --network my-net busybox sh
```



可以通过ping busybox1 或 busybox2验证通信情况



#### 4. DNS配置

默认不需要配置DNS，主机会将hostname、hosts、resolv.conf挂载到容器里

```
mount

/dev/disk/by-uuid/1fec...ebdf on /etc/hostname type ext4 ...

/dev/disk/by-uuid/1fec...ebdf on /etc/hosts type ext4 ...

tmpfs on /etc/resolv.conf type tmpfs ...
```

 

当然也可以手动配置全局的容器DNS，编辑/etc/docker/daemon.json

```
{

 "dns" : [

  "114.114.114.114",

  "8.8.8.8"

 ]

}
```

 

如果有针对容器定制需求的话可以在docker run的时候添加

-  -h HOSTNAME 或者 --hostname=HOSTNAME 设定容器的主机名，它会被写到容器内的 /etc/hostname 和 /etc/hosts
-  --dns=IP_ADDRESS 添加 DNS 服务器到容器的 /etc/resolv.conf 
-  --dns-search=DOMAIN 设定容器的搜索域，当设定搜索域为 .example.com 时，在搜索一个名为 host 的主机时，DNS 不仅搜索 host，还会搜索 host.example.com



### 5. 高级网络配置

主机和容器、容器和容器通信原理：

![截图.png](https://pcc.huitogo.club/%E6%88%AA%E5%9B%BE.png)

 

##### 5.1 容器网络配置

**Docker服务启动时**

- -b BRIDGE 或 --bridge=BRIDGE 指定容器挂载的网桥
- --bip=CIDR 定制 docker0 的掩码
- -H SOCKET... 或 --host=SOCKET... Docker 服务端接收命令的通道
- --icc=true|false 是否支持容器之间进行通信
- --ip-forward=true|false 请看下文容器之间的通信
- --iptables=true|false 是否允许 Docker 添加 iptables 规则
- --mtu=BYTES 容器网络中的 MTU



**容器启动时**

- -h HOSTNAME 或 --hostname=HOSTNAME 配置容器主机名
- --link=CONTAINER_NAME:ALIAS 添加到另一个容器的连接
- --net=bridge|none|container:NAME_or_ID|host 配置容器的桥接模式
- -p SPEC 或 --publish=SPEC 映射容器端口到宿主主机
- -P or --publish-all=true|false 映射容器所有端口到宿主主机



**两者都行**

- --dns=IP_ADDRESS... 使用指定的DNS服务器
- --dns-search=DOMAIN... 指定DNS搜索域



##### 5.2 网络工具

- pipework：帮助用户在比较复杂的场景中完成容器的连接。
- playground：提供完整的 Docker 容器网络拓扑管理的 Python库 (opens new window)，包括路由、NAT 防火墙；以及一些提供 HTTP SMTP POP IMAP Telnet SSH FTP 的服务器。