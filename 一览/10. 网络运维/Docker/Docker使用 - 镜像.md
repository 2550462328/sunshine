镜像和容器的可类比 Java中的类与实例的关系。镜像是静态的文件系统，独立的用户空间和存储系统，容器是运行时的镜像，是一个进程。

#### 1. 镜像拉取

```
docker pull [选项] [Docker Registry 地址[:端口号]/]仓库名[:标签]
```



#### 2. 镜像运行

```
docker run -it --rm ubuntu:18.04 bash
```

- -it：这是两个参数，一个是 -i：交互式操作，一个是 -t 终端。我们这里打算进入 bash 执行一些命令并查看返回结果，因此我们需要交互式终端。
- --rm：这个参数是说容器退出后随之将其删除。
- ubuntu:18.04：这是指用 ubuntu:18.04 镜像为基础来启动容器。
- bash：放在镜像名后的是 命令，这里我们希望有个交互式 Shell。



#### 3. 筛选镜像列表

```
# 列出所有镜像 不包括中间层镜像 
docker image ls 
# 包含中间层镜像  即公共依赖的镜像 没有名称和版本号 
docker image ls -a 
# 查看镜像摘要 
docker image ls --digests 
# 根据仓库名列举镜像 
docker image ls ubuntu 
# 指令仓库名和标签 
docker image ls ubuntu:18.04 
# 查询虚悬镜像：新旧发布镜像同名 旧的镜像名字会被剥夺 叫做虚悬镜像 
docker image ls -f dangling=true 
# 查看mongo:3.2之后建立的镜像 
docker image ls -f since=mongo:3.2 
# 查看mongo:3.2之前建立的镜像 
docker image ls -f before=mongo:3.2 
# 根据label筛选镜像 
docker image ls -f label=com.example.version=0.1
```



#### 4. 格式化镜像列表

```
# 只查询镜像ID 
docker image ls -q 
#只显示镜像ID 和 仓库名称 --- 参考GO的模板语法 
docker image ls --format "{{.ID}}: {{.Repository}}" 
# 使用表格样式 
docker image ls --format "table {{.ID}}\t{{.Repository}}\t{{.Tag}}"
```



#### 4. 删除镜像

```
# 删除虚悬镜像 
docker image prune 
# 根据镜像ID的最短可区分前缀删除 
docker image rm 501 
# 根据仓库名删除 
docker image rm centos 
# 根据镜像摘要删除镜像 
docker image rm node@sha256:b4f0e0bdeb578043c1ea6862f0d40cc4afe32a4a582f3be235a3b164422be228 
# 根据docker image ls 的结果删除镜像 
docker image rm $(docker image ls -q redis)
```



#### 5. 查看镜像

```
# 查看镜像、容器、数据卷占用空间 
docker system df
```



#### 6. 多阶段构建

多阶段构建 的好处是 其他镜像可以复用某一层，或者说我们docker run的时候也可以按层启动，总的来说灵活度大大增强

 

比如我们要构建Laravel 镜像

第一阶段：构建前端

```
ROM node:alpine as frontend


COPY package.json /app/


RUN set -x ; cd /app \

   && npm install --registry=https://registry.npmmirror.com


COPY webpack.mix.js webpack.config.js tailwind.config.js /app/

COPY resources/ /app/resources/


RUN set -x ; cd /app \

   && touch artisan \

   && mkdir -p public \

   && npm run production
```

 

第二阶段：安装 Composer 依赖

```
FROM composer as composer


COPY database/ /app/database/

COPY composer.json composer.lock /app/


RUN set -x ; cd /app \

   && composer config -g repo.packagist composer https://mirrors.aliyun.com/composer/ \

   && composer install \

     --ignore-platform-reqs \

     --no-interaction \

     --no-plugins \

     --no-scripts \

     --prefer-dist
```

 

第三阶段：整合文件

```
FROM php:7.4-fpm-alpine as laravel


ARG LARAVEL_PATH=/app/laravel


COPY --from=composer /app/vendor/ ${LARAVEL_PATH}/vendor/

COPY . ${LARAVEL_PATH}

COPY --from=frontend /app/public/js/ ${LARAVEL_PATH}/public/js/

COPY --from=frontend /app/public/css/ ${LARAVEL_PATH}/public/css/

COPY --from=frontend /app/public/mix-manifest.json ${LARAVEL_PATH}/public/mix-manifest.json


RUN set -x ; cd ${LARAVEL_PATH} \

   && mkdir -p storage \

   && mkdir -p storage/framework/cache \

   && mkdir -p storage/framework/sessions \

   && mkdir -p storage/framework/testing \

   && mkdir -p storage/framework/views \

   && mkdir -p storage/logs \

   && chmod -R 777 storage \

   && php artisan package:discover
```

 

第四阶段：构建nginx镜像

```
FROM nginx:alpine as nginx


ARG LARAVEL_PATH=/app/laravel


COPY laravel.conf /etc/nginx/conf.d/

COPY --from=laravel ${LARAVEL_PATH}/public ${LARAVEL_PATH}/public

 
构建镜像

# 构建laravel镜像

docker build -t my/laravel --target=laravel .

# 构建nginx镜像

docker build -t my/laravel --target=nginx .
```



#### 6. 多版本构建

我们在当前操作系统上构建和提交到docker hub的镜像也只能被和当前一样的操作系统所拉取，通常情况下我们可以在版本号里说明镜像支持的操作系统

但有更简便的方法，使用manifest说明和区分

 

创建manifest --- create

```
# $ docker manifest create MANIFEST_LIST MANIFEST [MANIFEST...]

$ docker manifest create username/test \

   username/x8664-test \

   username/arm64v8-test
```

 

设置manifest --- annotate

```
# $ docker manifest annotate [OPTIONS] MANIFEST_LIST MANIFEST

$ docker manifest annotate username/test \

   username/x8664-test \

   --os linux --arch x86_64

 

$ docker manifest annotate username/test \

   username/arm64v8-test \

   --os linux --arch arm64 --variant v8
```

 

查看manifest --- inspect

```
docker manifest inspect username/test
```

 

推送manifest --- push

```
docker manifest push username/test
```

