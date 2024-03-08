#### 1. 挂载数据卷

```
# 创建数据卷

docker volume create my-vol

# 数据卷列表

docker volume ls

# 查看数据卷

docker volume inspect my-vol

# 启动挂在数据卷的容器

docker run -d -P \

  --name web \

  --mount source=my-vol,target=/usr/share/nginx/html \

  nginx:alpine

# 删除数据卷

docker volume rm my-vol

# 清理无主的数据卷

docker volume prune
```



#### 2. 挂载主机目录

```
# 直接挂在本机目录到容器目录

docker run -d -P \

  --name web \

  # -v /src/webapp:/usr/share/nginx/html \

  --mount type=bind,source=/src/webapp,target=/usr/share/nginx/html \

  nginx:alpine

 

# 设置挂载目录为只读 readonly

docker run -d -P \

  --name web \

  # -v /src/webapp:/usr/share/nginx/html:ro \

  --mount type=bind,source=/src/webapp,target=/usr/share/nginx/html,readonly \

  nginx:alpine

 

# 仅挂载单个文件

docker run --rm -it \

  # -v $HOME/.bash_history:/root/.bash_history \

  --mount type=bind,source=$HOME/.bash_history,target=/root/.bash_history \

  ubuntu:18.04 \

  bash
```

