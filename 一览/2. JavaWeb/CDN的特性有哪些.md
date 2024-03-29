
CDN ( Content Delivery Network ， 内容分发网络）指基于部署在各地的机房服务器， 通过中心平台的负载均衡、内容分发、调度的能力，使用户就近获取所需内容，降低网络延迟，提升用户访问的响应速度和体验度 。



**CDN 的关键技术**

CDN 的关键技术包括内容发布、内容路由、内容交换和性能管理，具体如下 。

- **内容发布** ： 借助建立索引 、缓存 、流分裂、组播等技术，将内容发布到网络上距 离用户最近的中心机房。
- **内容路由**： 通过内容路由器中的重定向（ DNS ）机制，在多个中心机房的 服务器 上负载均衡用户的请求，使用户从最近的中心机房获取数据 。
- **内容交换** ： 根据内容的可用性 、 服务器的可用性及用户的背景，在缓存服务器上利用应用层交换、流分裂、重定 向等技术，智能地平衡负载流量 。
- **性能管理**：通过内部和外部监控系统，获取网络部件的信息，测量内容发布的端 到端性能（包丢失 、 延时 、 平均带宽 、 启动时间 、 帧速率等），保证网络处于最佳运行状态 。



**CDN 的主要特点**

- 本地缓存（ Cache ）加速 ： 将用户经常访问的数据（尤其静态数据）缓存在本地 ， 以提升系统的响应速度和稳定性 。
- 镜像服务 ： 消除不同运营商之间的网络差异，实现跨运营商的网络加速，保证不 同运营商网络中的用户都能得到良好的网络体验 。
- 远程加速 ： 利用 DNS 负载均衡技术为用户选择服务质量最优的服务器，加快用 户远程访问 的速度 。
- 带宽优化 ：自动生成服务器的远程镜像缓存服务器，远程用户在访问时从就近的 缓存服务器上读取数据，减少远程访问的带宽，分担网络流量，并降低原站点的 Web 服务器负载等 。
- 集群抗攻击：通过网络安全技术和 CDN 之间的智能冗余机制，可以有效减少网 络攻击对网站的影响 。



#### 1. 内容分发系统

将用户请求的数据分发到就近的各个中心机房，以保障为用户提供快速、高效的内 容服务 。 缓存的内容包括静态图片、视频 、文本、用户最近访 问 的 JSON 数据等 。 缓存 的技术包括 内存环境、分布式缓存、本地文件缓存等 。 缓存的策略主要考虑缓存更新、 缓存淘汰机制 。



#### 2. 负载均衡系统

负载均衡系统是整个 CDN 系统的核心，负载均衡根据当前网络的流量分布、各中心 机房服务器的负载和用户请求的特点将用户的请求负载到不 同的中心机房或不同的服务 器上，以保障用户 内 容访问的流畅性 。



负载均衡系统包括全局负载均衡（ GSLB ）和本地 负载均衡（ SLB ）。

1. 全局负载均衡主要指跨机房的负载均衡，通过 DNS 解析或者应用层重定向技术 将用户的请求负载到就近的中心机房上 。
2. 本地负载均衡主要指机房内部的负载均衡，一般通过缓存服务器，基于 LVS,、Nginx 、服务网关等技术实现用户访问的负载。



#### 3. 管理系统

管理系统分为运营管理和网络管理子系统 。 网络管理系统主要对整个 CDN 网络资源 的运行状态进行实时监控和管理 。 运营管理指对 CDN 日常运维业务的管理，包括用户管 理、资源管理、流量计费和流量限流等 。