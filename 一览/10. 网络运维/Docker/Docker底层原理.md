#### 1.  虚拟化技术

首先，Docker **容器虚拟化技术**为基础的软件，那么什么是虚拟化技术呢？



简单点来说，虚拟化技术可以这样定义：

虚拟化技术是一种资源管理技术，是将计算机的各种[实体资源](https://zh.wikipedia.org/wiki/資源_(計算機科學))（[CPU](https://zh.wikipedia.org/wiki/CPU)、[内存](https://zh.wikipedia.org/wiki/内存)、[磁盘空间](https://zh.wikipedia.org/wiki/磁盘空间)、[网络适配器](https://zh.wikipedia.org/wiki/網路適配器)等），予以抽象、转换后呈现出来并可供分割、组合为一个或多个电脑配置环境。由此，打破实体结构间的不可切割的障碍，使用户可以比原本的配置更好的方式来应用这些电脑硬件资源。这些资源的新虚拟部分是不受现有资源的架设方式，地域或物理配置所限制。一般所指的虚拟化资源包括计算能力和数据存储。



#### 2.  基于 LXC 虚拟容器技术

Docker 最初是基于 LXC（Linux container- Linux 容器）虚拟容器技术的，从 0.7 版本以后开始去除 LXC。

LXC，其名称来自 Linux 软件容器（Linux Containers）的缩写，一种操作系统层虚拟化（Operating system–level virtualization）技术，为 Linux 内核容器功能的一个用户空间接口。它将应用软件系统打包成一个软件容器（Container），内含应用软件本身的代码，以及所需要的操作系统核心和库。通过统一的名字空间和共用 API 来分配不同软件容器的可用硬件资源，创造出应用程序的独立沙箱运行环境，使得 Linux 用户可以容易的创建和管理系统或应用容器。

LXC 技术主要是借助 Linux 内核中提供的 CGroup 功能和 name space 来实现的，通过 LXC 可以为软件提供一个独立的操作系统运行环境。



**cgroup 和 namespace 介绍：**

**namespace 是 Linux 内核用来隔离内核资源的方式**。 通过 namespace 可以让一些进程只能看到与自己相关的一部分资源，而另外一些进程也只能看到与它们自己相关的资源，这两拨进程根本就感觉不到对方的存在。具体的实现方式是把一个或多个进程的相关资源指定在同一个 namespace 中。Linux namespaces 是对全局系统资源的一种封装隔离，使得处于不同 namespace 的进程拥有独立的全局系统资源，改变一个 namespace 中的系统资源只会影响当前 namespace 里的进程，对其他 namespace 中的进程没有影响。

**CGroup 是 Control Groups 的缩写，是 Linux 内核提供的一种可以限制、记录、隔离进程组 (process groups) 所使用的物力资源 (如 cpu memory i/o 等等) 的机制**。



**cgroup 和 namespace 两者对比：**

两者都是将进程进行分组，但是两者的作用还是有本质区别。namespace 是为了隔离进程组之间的资源，而 cgroup 是为了对一组进程进行统一的资源监控和限制。



#### 3. 【Docker 基于 C/S架构】

![截图.png](https://pcc.huitogo.club/%E6%88%AA%E5%9B%BE1.png)

 

##### 3.1 底层支持

Docker 底层的核心技术包括 Linux 上的命名空间（Namespaces）、控制组（Control groups）、Union 文件系统（Union file systems）和容器格式（Container format）。

 

传统虚拟机需要对内存、CPU、网络IO、硬盘IO、存储空间等的限制，还要实现文件系统、网络、PID、UID、IPC等等的相互隔离。会出现系统资源的浪费。

容器化基于Linux系统对于命名空间完善支持，让某些进程在彼此隔离的命名空间中运行。大家虽然都共用一个内核和某些运行时环境（例如一些系统命令和系统库），但是彼此却看不到，都以为系统中只有自己的存在。这种机制就是容器（Container），利用命名空间来做权限的隔离控制，利用 cgroups 来做资源分配。

 

**1）命名空间**

l pid 命名空间：不同用户的进程就是通过 pid 命名空间隔离开的，且不同命名空间中可以有相同 pid。

l net命名空间：有了 pid 命名空间，每个命名空间中的 pid 能够相互隔离，但是网络端口还是共享 host 的端口。网络隔离是通过 net 命名空间实现的， 每个 net 命名空间有独立的 网络设备，IP 地址，路由表，/proc/net 目录。这样每个容器的网络就能隔离开来。

l ipc命名空间：容器中进程交互还是采用了 Linux 常见的进程间交互方法(interprocess communication - IPC)， 包括信号量、消息队列和共享内存等。然而同 VM 不同的是，容器的进程间交互实际上还是 host 上具有相同 pid 命名空间中的进程间交互，因此需要在 IPC 资源申请时加入命名空间信息。

l mnt命名空间：类似 chroot，将一个进程放到一个特定的目录执行。mnt 命名空间允许不同命名空间的进程看到的文件结构不同，这样每个命名空间 中的进程所看到的文件目录就被隔离开了。

l uts命名空间：UTS("UNIX Time-sharing System") 命名空间允许每个容器拥有独立的 hostname 和 domain name， 使其在网络上可以被视作一个独立的节点而非 主机上的一个进程。

l user命令空间：每个容器可以有不同的用户和组 id， 也就是说可以在容器内用容器内部的用户执行程序而非主机上的用户。

 

**2）控制组**

控制组可以提供对容器的内存、CPU、磁盘 IO 等资源的限制和审计管理。

只有能控制分配到容器的资源，才能避免当多个容器同时运行时的对系统资源的竞争。

 

**3）联合文件系统**

联合文件系统（UnionFS (opens new window)）是一种分层、轻量级并且高性能的文件系统，它支持对文件系统的修改作为一次提交来一层层的叠加，同时可以将不同目录挂载到同一个虚拟文件系统下(unite several directories into a single virtual filesystem)。

 

联合文件系统是 Docker 镜像的基础。镜像可以通过分层来进行继承，基于基础镜像（没有父镜像），可以制作各种具体的应用镜像。

 

另外，不同 Docker 容器就可以共享一些基础的文件系统层，同时再加上自己独有的改动层，大大提高了存储的效率。

 

Docker 目前支持的联合文件系统包括 OverlayFS, AUFS, Btrfs, VFS, ZFS 和 Device Mapper。默认推荐使用OverlayFS作为存储驱动。

 

**4）容器格式**

最初，Docker 采用了 LXC 中的容器格式。从 0.7 版本以后开始去除 LXC，转而使用自行开发的 libcontainer (opens new window)，从 1.11 开始，则进一步演进为使用 runC (opens new window)和 containerd (opens new window)。

 

##### 3.2 网络通信原理

**1）基于veth pair**

首先，要实现网络通信，机器需要至少一个网络接口（物理接口或虚拟接口）来收发数据包；此外，如果不同子网之间要进行通信，需要路由机制。

 

Docker 中的网络接口默认都是虚拟的接口。虚拟接口的优势之一是转发效率较高。 Linux 通过在内核中进行数据复制来实现虚拟接口之间的数据转发，发送接口的发送缓存中的数据包被直接复制到接收接口的接收缓存中。对于本地系统和容器内系统看来就像是一个正常的以太网卡，只是它不需要真正同外部网络设备通信，速度要快很多。

 

Docker 容器网络就利用了这项技术。它在本地主机和容器内分别创建一个虚拟接口，并让它们彼此连通（这样的一对接口叫做 veth pair）。

 

**2）通信流程**

Docker 创建一个容器的时候，会执行如下操作：

1. 创建一对虚拟接口，分别放到本地主机和新容器中；
2. 本地主机一端桥接到默认的 docker0 或指定网桥上，并具有一个唯一的名字，如 veth65f9；
3. 容器一端放到新容器中，并修改名字作为 eth0，这个接口只在容器的命名空间可见；
4. 从网桥可用地址段中获取一个空闲地址分配给容器的 eth0，并配置默认路由到桥接网卡 veth65f9。



完成这些之后，容器就可以使用 eth0 虚拟网卡来连接其他容器和其他网络。

 

可以在 docker run 的时候通过 --net 参数来指定容器的网络配置，有4个可选值：

-  --net=bridge 这个是默认值，连接到默认的网桥。
-  --net=host 告诉 Docker 不要将容器网络放到隔离的命名空间中，即不要容器化容器内的网络。此时容器使用本地主机的网络，它拥有完全的本地主机接口访问权限。容器进程可以跟主机其它 root 进程一样可以打开低范围的端口，可以访问本地网络服务比如 D-bus，还可以让容器做一些影响整个主机系统的事情，比如重启主机。因此使用这个选项的时候要非常小心。如果进一步的使用 --privileged=true，容器会被允许直接配置主机的网络堆栈。
-  --net=container:NAME_or_ID 让 Docker 将新建容器的进程放到一个已存在容器的网络栈中，新容器进程有自己的文件系统、进程列表和资源限制，但会和已存在的容器共享 IP 地址和端口等网络资源，两者进程可以直接通过 lo 环回接口通信。
-  --net=none 让 Docker 将新容器放到隔离的网络栈中，但是不进行网络配置。之后，用户可以自己进行配置。

 

**3）自定义网络**

首先，启动一个 /bin/bash 容器，指定 --net=none 参数。

```
$ docker run -i -t --rm --net=none base /bin/bash

root@63f36fc01b5f:/#
```



在本地主机查找容器的进程 id，并为它创建网络命名空间。

```
$ docker inspect -f '{{.State.Pid}}' 63f36fc01b5f

2778

$ pid=2778

$ sudo mkdir -p /var/run/netns

$ sudo ln -s /proc/$pid/ns/net /var/run/netns/$pid
```



检查桥接网卡的 IP 和子网掩码信息。

```
$ ip addr show docker0

21: docker0: ...

inet 172.17.42.1/16 scope global docker0

...
```



创建一对 “veth pair” 接口 A 和 B，绑定 A 到网桥 docker0，并启用它

```
$ sudo ip link add A type veth peer name B

$ sudo brctl addif docker0 A

$ sudo ip link set A up
```



将B放到容器的网络命名空间，命名为 eth0，启动它并配置一个可用 IP（桥接网段）和默认网关。

```
$ sudo ip link set B netns $pid

$ sudo ip netns exec $pid ip link set dev B name eth0

$ sudo ip netns exec $pid ip link set eth0 up

$ sudo ip netns exec $pid ip addr add 172.17.42.99/16 dev eth0

$ sudo ip netns exec $pid ip route add default via 172.17.42.1
```



当容器结束后，Docker 会清空容器，容器内的 eth0 会随网络命名空间一起被清除，A 接口也被自动从 docker0 卸载。

 

此外，用户可以使用 ip netns exec 命令来在指定网络命名空间中进行配置，从而配置容器内的网络。