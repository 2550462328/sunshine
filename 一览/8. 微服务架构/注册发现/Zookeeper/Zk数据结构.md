ZooKeeper 数据模型采用层次化的多叉树形结构，每个节点上都可以存储数据，这些数据可以是数字、字符串或者是二级制序列。



类似下图

![img](http://pcc.huitogo.club/05442f8ef446afec15d5cd0cabbc18c7)



其中znode是 ZooKeeper 中数据的最小单元。你要存放的数据就放在上面。

它的类型分为4大类

1. **持久（PERSISTENT）节点** ：一旦创建就一直存在即使 ZooKeeper 集群宕机，直到将其删除。
2. **临时（EPHEMERAL）节点** ：临时节点的生命周期是与 客户端会话（session） 绑定的，会话消失则节点消失 。并且，临时节点只能做叶子节点 ，不能创建子节点。
3. **持久顺序（PERSISTENT_SEQUENTIAL）节点** ：除了具有持久（PERSISTENT）节点的特性之外， 子节点的名称还具有顺序性。比如 /node1/app0000000001 、/node1/app0000000002 。
4. **临时顺序（EPHEMERAL_SEQUENTIAL）节点** ：除了具备临时（EPHEMERAL）节点的特性之外，子节点的名称还具有顺序性。



但是还是需要强调的是：**ZooKeeper 主要是用来协调服务的，而不是用来存储业务数据的，所以不要放比较大的数据在 znode 上，ZooKeeper 给出的上限是每个结点的数据大小最大是 1M。**



**znode数据结构**

每个 znode 由 2 部分组成:

- stat ：状态信息
- data ： 节点存放的数据的具体内容



Stat 类中包含了一个数据节点的所有状态信息的字段，包括事务 ID-cZxid、节点创建时间-ctime 和子节点个数-numChildren 等等。

![img](http://pcc.huitogo.club/55f15de32e6a77b8635bb41168bbab12)



**znode权限**

对于 znode 操作的权限，ZooKeeper 提供了以下 5 种：

- CREATE : 能创建子节点
- READ ：能获取节点数据和列出其子节点
- WRITE : 能设置/更新节点数据
- DELETE : 能删除子节点
- ADMIN : 能设置节点 ACL 的权限



其中尤其需要注意的是，CREATE 和 DELETE 这两种权限都是针对 子节点 的权限控制。



对于身份认证，提供了以下几种方式：

- world ： 默认方式，所有用户都可无条件访问。
- auth :不使用任何 id，代表任何已认证的用户。
- digest :用户名:密码认证方式： username:password 。
- ip : 对指定 ip 进行限制。