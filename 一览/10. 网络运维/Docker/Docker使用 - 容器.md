#### 1. 启动容器

```
docker run ubuntu:18.04 /bin/echo 'Hello world'

docker run -t -i ubuntu:18.04 /bin/bash

# 启动已停止的容器

docker container start

# 重启容器

docker container restart

# 后台启动 -d 这里不会导致容器主进程退出

docker run -d ubuntu:18.04 /bin/sh
```

 

docker run的过程

1. 检查本地是否存在指定的镜像，不存在就从 [registry](https://vuepress.mirror.docker-practice.com/repository/) 下载
2. 利用镜像创建并启动一个容器
3. 分配一个文件系统，并在只读的镜像层外面挂载一层可读写层
4. 从宿主主机配置的网桥接口中桥接一个虚拟接口到容器中去
5. 从地址池配置一个 ip 地址给容器
6. 执行用户指定的应用程序
7. 执行完毕后容器被终止

 

#### 2. 容器列表

```
docker container ls 

# 查看已终止的容器

docker container ls -a
```



#### 3. 查看容器

```
# 查看容器日志

docker container logs [container ID or NAMES]

# 进入容器  在容器中exit会导致容器终止

docker attach [container ID or NAMES]

# 进入容器  在容器中exit不会导致容器终止

docker exec [container ID or NAMES]

# 打开执行结果 -t打开终端 -i打开结果

docker exec -it [container ID or NAMES]
```



#### 4. 终止容器

```
docker container stop
```



#### 5. 迁移容器

```
# 容器导出

docker export 7691a814370e > ubuntu.tar

# 容器导入

cat ubuntu.tar | docker import - test/ubuntu:v1.0

# 导入容器快照链接

docker import http://example.com/exampleimage.tgz example/imagerepo
```

 

docker import 容器快照 和 docker load 镜像 的区别在于，容器快照仅保留当时的快照状态，所以体积较小，而镜像则保留完整记录，包括历史记录、元数据信息等，体积较大



#### 6. 删除容器

```
# 删除终止状态的容器

docker container rm trusting_newton

# 强制删除运行状态的容器

docker -f container rm trusting_newton

# 清理所有终止状态的容器

docker container prune
```

