**1. 本地的镜像文件都存放在哪里？**

答：与 Docker 相关的本地资源默认存放在 /var/lib/docker/ 目录下，以 overlay2 文件系统为例，其中 containers 目录存放容器信息，image 目录存放镜像信息，overlay2 目录下存放具体的镜像层文件。Docker的配置文件默认存放/etc/docker/daemon.json。

 

**2. 如何修改默认的存储目录**

答：Docker 的默认存储位置是 /var/lib/docker，如果希望将 Docker 的本地文件存储到其他分区，可以使用 Linux 软连接的方式来完成，或者在启动 [[daemon 时通过 -g 参数指定，或者修改配置文件 /etc/docker/daemon.json 的 "data-root" 项 。可以使用 docker system info | grep "Root Dir" 查看当前使用的存储位置。

 

例如，如下操作将默认存储位置迁移到 /storage/docker。

```
service docker stop

cd /var/lib/

mv docker /storage/

ln -s /storage/docker/ docker

ls -la docker

lrwxrwxrwx. 1 root root 15 11月 17 13:43 docker -> /storage/docker

service docker start
```

 

**3. 构建 Docker 镜像应该遵循哪些原则？**

答：整体原则上，尽量保持镜像功能的明确和内容的精简，要点包括

1. 尽量选取满足需求但较小的基础系统镜像，例如大部分时候可以选择 alpine 镜像，仅有不足六兆大小；
2. 清理编译生成文件、安装包的缓存等临时文件；
3. 安装各个软件时候要指定准确的版本号，并避免引入不需要的依赖；
4. 从安全角度考虑，应用要尽量使用系统的库和依赖；
5. 如果安装应用时候需要配置一些特殊的环境变量，在安装后要还原不需要保持的变量值；
6. 使用 Dockerfile 创建镜像时候要添加 .dockerignore 文件或使用干净的工作目录。

 

**4. 碰到网络问题，无法 pull 镜像，命令行指定 http_proxy 无效？**

答：在 Docker 配置文件中添加 export http_proxy="http://<PROXY_HOST>:<PROXY_PORT>"，之后重启 Docker 服务即可。

 

**5. 如何获取某个容器的 PID 信息？**

答：可以使用

```
docker inspect --format '{{ .State.Pid }}' <CONTAINER ID or NAME>
```

 

**6. 如何获取某个容器的 IP 地址？**

答：可以使用

```
docker inspect --format '{{ .NetworkSettings.IPAddress }}' <CONTAINER ID or NAME>
```

 

**7. 如何给容器指定一个固定 IP 地址，而不是每次重启容器 IP 地址都会变？**

答：使用以下命令启动容器可以使容器 IP 固定不变

```
$ docker network create -d bridge --subnet 172.25.0.0/16 my-net

$ docker run --network=my-net --ip=172.25.3.3 -itd --name=my-container busybox
```

 

***如何临时退出一个正在交互的容器的终端，而不终止它？\***

答：按 Ctrl-p Ctrl-q。如果按 Ctrl-c 往往会让容器内应用进程终止，进而会终止容器。

 

**8. 如何控制容器占用系统资源（CPU、内存）的份额？**

答：在使用 docker create 命令创建容器或使用 docker run 创建并启动容器的时候，可以使用 -c|--cpu-shares[=0] 参数来调整容器使用 CPU 的权重；使用 -m|--memory[=MEMORY] 参数来调整容器使用内存的大小。

 

**9. Docker 与 LXC（Linux Container）有何不同？**

答：LXC 利用 Linux 上相关技术实现了容器。Docker 则在如下的几个方面进行了改进：

-  移植性：通过抽象容器配置，容器可以实现从一个平台移植到另一个平台；镜像系统：基于 OverlayFS 的镜像系统为容器的分发带来了很多的便利，同时共同的镜像层只需要存储一份，实现高效率的存储；版本管理：类似于Git的版本管理理念，用户可以更方便的创建、管理镜像文件；
- 仓库系统：仓库系统大大降低了镜像的分发和管理的成本；
- 周边工具：各种现有工具（配置管理、云平台）对 Docker 的支持，以及基于 Docker的 PaaS、CI 等系统，让 Docker 的应用更加方便和多样化。

 

**10. 如何将一台宿主主机的 Docker 环境迁移到另外一台宿主主机**？

答：停止 Docker 服务。将整个 Docker 存储文件夹复制到另外一台宿主主机，然后调整另外一台宿主主机的配置即可。

 

**11. 如何获取容器绑定到本地那个 veth 接口上？**

答：Docker 容器启动后，会通过 veth 接口对连接到本地网桥，veth 接口命名跟容器命名毫无关系，十分难以找到对应关系。

 

最简单的一种方式是通过查看接口的索引号，在容器中执行 ip a 命令，查看到本地接口最前面的接口索引号，如 205，将此值加上 1，即 206，然后在本地主机执行 ip a 命令，查找接口索引号为 206 的接口，两者即为连接的 veth 接口对。