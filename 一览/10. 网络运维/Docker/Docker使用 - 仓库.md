#### 1. 公共仓库 docker hub

```
# 搜索镜像

docker search centos

# 筛选收藏量大于N的镜像

docker search centos --filter=stars=N

# 拉取镜像

docker pull centos

# 预备提交 - 给本地的镜像打TAG

docker tag ubuntu:18.04 username/ubuntu:18.04

# 提交镜像

docker push username/ubuntu:18.04
```



#### 2. 私有仓库

```
# 基于docker registry镜像来搭建

docker run -d -p 5000:5000 --restart=always --name registry registry

# 修改默认镜像上传文件地址

docker run -d \

  -p 5000:5000 \

  -v /opt/data/registry:/var/lib/registry \

  registry

# 查看私有仓库的镜像

curl 127.0.0.1:5000/v2/_catalog
```



注意：私有仓库不接受非https的请求，如果必须是http的，可以给它开白名单



编辑 /etc/docker/daemon.json (没有则新增)

```
{

 "registry-mirror": [

  "https://hub-mirror.c.163.com",

  "https://mirror.baidubce.com"

 ],

 "insecure-registries": [

  "192.168.199.100:5000"

 ]

}
```

