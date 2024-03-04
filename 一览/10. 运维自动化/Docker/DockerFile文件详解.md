#### 1. COPY

```
COPY package.json /usr/src/app/

# 使用通配符匹配

COPY hom?.txt /mydir
```

 

注意：这种拷贝会保留文件的元数据，比如读、写、执行权限、文件变更时间等。

如果要修改文件的权限，可以使用--chown修改

COPY --chown=55:mygroup files* /mydir/



#### 2. ADD

```
ADD package.json /usr/src/app/
```



基本和COPY类似，但是它会有额外的解析动作，比如源地址是URL、压缩文件它会进行下载和解压

建议使用：复制文件用COPY + 解压文件用ADD



#### 3. CMD

```
CMD echo $HOME

# 或

CMD [ "sh", "-c", "echo $HOME" ]
```



CMD是用来用于指定默认的容器主进程的启动命令

注意：**启动的容器进程只能前台启动**，因为容器不是虚拟机，它只是一个进程，如果主进程退出了（后台启动）那么容器也就没有存在的意义了。



#### 4. ENTRYPOINT

```
ENTRYPOINT [ "curl", "-s", "http://myip.ipip.net" ]
```

 

跟CMD功能类似，但是它可以替换CMD的默认值作为参数加入到执行指令中，简而言之就是允许在docker run的时候添加参数

 

例如打印网络ip的时候输出HTTP头

```
# DockerFile

FROM ubuntu:18.04

RUN apt-get update \

  && apt-get install -y curl \

  && rm -rf /var/lib/apt/lists/*

ENTRYPOINT [ "curl", "-s", "http://myip.ipip.net" ]


# run

docker run myip -i


# 上述效果等同于用CMD的情况下

docker run myip curl -s http://myip.ipip.net -i
```

 

在redis的镜像中也会根据获取启动id

```
FROM alpine:3.4

...

RUN addgroup -S redis && adduser -S -G redis redis

...

ENTRYPOINT ["docker-entrypoint.sh"]

EXPOSE 6379

CMD [ "redis-server" ]

# docker-entrypoint.sh

...

# allow the container to be started with `--user`

if [ "$1" = 'redis-server' -a "$(id -u)" = '0' ]; then

find . \! -user redis -exec chown redis '{}' +

exec gosu redis "$0" "$@"

fi

exec "$@"
```

该脚本的内容就是根据 CMD 的内容来判断，如果是 redis-server 的话，则切换到 redis 用户身份启动服务器，否则依旧使用 root 身份执行。



#### 5. ENV

```
ENV NODE_VERSION 7.2.0
```

设置环境变量，便于使用和扩展，**可以在容器运行时使用**



#### 6. ARG

```
ARG DOCKER_USERNAME=library
```

 

使用和ENV类似，需要注意的是

- ARG的参数只能作用于它下面的最近一条运行指令 比如From、CMD、RUN，如果多阶段需要重复指定

```
ARG DOCKER_USERNAME=library

FROM ${DOCKER_USERNAME}/alpine

# 在FROM 之后使用变量，必须在每个阶段分别指定

ARG DOCKER_USERNAME=library

RUN set -x ; echo ${DOCKER_USERNAME}
```

 

- ARG的参数只是默认值，可以在docker build时指定覆盖

```
docker build --build-arg <参数名>=<值>
```

 

- ARG的环境变量在容器运行时会访问不到的

  

#### 7. VOLUME

原则上在容器启动时不能写入存储层，对于数据库类的应用需要保存动态数据，应该把文件保存在卷中，为了防止用户忘记将动态数据挂载到卷上，我们可以针对指定的路径写入声明匿名卷

```
# 针对/data的写入挂载到匿名卷

VOLUME /data

# 可以docker run的时候覆盖

docker run -d -v mydata:/data xxxx
```



#### 8. EXPOSE

```
EXPOSE 8090
```

注意：EXPOSE只是声明端口，具体容器映射端口可以在docker run -p指定，如果使用随机端口容器会选择EXPOSE声明的端口。



#### 9. WORKDIR

在shell脚本编写中我们可能会使用下面语法切换工作空间

```
RUN cd /app

RUN echo "hello" > world.txt
```

 

但在Doker的多层存储空间的模型下，很明显这样是不生效的，如果我们要指定工作空间可以这么写

```
WORKDIR /app

RUN echo "hello" > world.txt

 

#或者跳转多次 每次会取决于上一次的WORKDIR

WORKDIR /a

WORKDIR b

WORKDIR c
```



#### 10. USER

```
RUN groupadd -r redis && useradd -r -g redis redis

USER redis

RUN [ "redis-server" ]
```

指定用户，作用于后面的指令，用户需要提前建立

 

但如果是执行期间需要切换用户，比如这个指令需要用root权限执行，但是执行中需要切换到elastic用户，可以使用gosu命令

```
# 建立 redis 用户，并使用 gosu 换另一个用户执行命令

RUN groupadd -r redis && useradd -r -g redis redis

# 下载 gosu

RUN wget -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/1.12/gosu-amd64" \

  && chmod +x /usr/local/bin/gosu \

  && gosu nobody true

# 设置 CMD，并以另外的用户执行

CMD [ "exec", "gosu", "redis", "redis-server" ]
```



#### 11. HEALTHCHECK

容器健康检查，默认只要容器的主进程不退出即认为是正常运行，我们可以添加自定义指令判断容器是否正常运行

```
FROM nginx

RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

HEALTHCHECK --interval=5s --timeout=3s \

 CMD curl -fs http://localhost/ || exit 1
```

 

可以通过docker inspect查看健康监控日志

```
docker inspect --format '{{json .State.Health}}' web | python -m json.tool
```



#### 12. LABEL

给镜像添加一些元数据，格式是键值对

```
LABEL org.opencontainers.image.authors="yeasy"
```



#### 13. SHELL

替代CMD、RUN、ENTRYPOINT默认的shell执行脚本

```
# 默认脚本

SHELL ["/bin/sh", "-c"]

RUN lll ; ls

 

SHELL ["/bin/sh", "-cex"]

RUN lll ; ls
```



#### 14. ONBUILD

配置基础镜像

比如我们部署集群的时候，每台服务的Dockerfile都是差不太多，如果每个服务都复制一份Dockerfile改改，就不利于后期维护，所有需要针对公共的部分提取到基础镜像里，然后我们在第二层直接引用

 

基础镜像my-node

```
FROM node:slim

RUN mkdir /app

WORKDIR /app

ONBUILD COPY ./package.json /app

ONBUILD RUN [ "npm", "install" ]

ONBUILD COPY . /app/

CMD [ "npm", "start" ]
```

 

使用

```
FROM my-node
```

