# zk结构

ZooKeeper是一个开源的分布式服务框架，它是Apache Hadoop项目的一个子项目，主要用来解决分布式应用场景中存在的一些问题，如：统一命名服务、状态同步服务、集群管理、分布式应用配置管理等，它支持Standalone模式和分布式模式，在分布式模式下，能够为分布式应用提供高性能和可靠地协调服务，而且使用ZooKeeper可以大大简化分布式协调服务的实现，为开发分布式应用极大地降低了成本。 

![](/assets/zkservice.jpg)

ZooKeeper集群由一组Server节点组成，这一组Server节点中存在一个角色为Leader的节点，其他节点都为Follower。当客户端Client连接到ZooKeeper集群，并且执行写请求时，这些请求会被发送到Leader节点上，然后Leader节点上数据变更会同步到集群中其他的Follower节点。

 Leader节点在接收到数据变更请求后，首先将变更写入本地磁盘，以作恢复之用。当所有的写请求持久化到磁盘以后，才会将变更应用到内存中。

 ZooKeeper使用了一种自定义的原子消息协议，在消息层的这种原子特性，保证了整个协调系统中的节点数据或状态的一致性。Follower基于这种消息协议能够保证本地的ZooKeeper数据与Leader节点同步，然后基于本地的存储来独立地对外提供服务。

 当一个Leader节点发生故障失效时，失败故障是快速响应的，消息层负责重新选择一个Leader，继续作为协调服务集群的中心，处理客户端写请求，并将ZooKeeper协调系统的数据变更同步（广播）到其他的Follower节点。



# 数据模型

ZooKeeper有一个分层的命名空间，结构类似文件系统的目录结构，非常简单而直观。其中，ZNode是最重要的概念，前面我们已经描述过。另外，有ZNode有关的还包括Watches、ACL、临时节点、序列节点（Sequence Node）。 

  ZNode结构

ZooKeeper中使用Zxid（ZooKeeper Transaction Id）来表示每次节点数据变更，一个Zxid与一个时间戳对应，所以多个不同的变更对应的事务是有序的。下面是ZNode的组成结构，引用文档如下所示：

* czxid – The zxid of the change that caused this znode to be created.
* mzxid – The zxid of the change that last modified this znode.
* ctime – The time in milliseconds from epoch when this znode was created.
* mtime – The time in milliseconds from epoch when this znode was last modified.
* version – The number of changes to the data of this znode.
* cversion – The number of changes to the children of this znode.
* aversion – The number of changes to the ACL of this znode.
* ephemeralOwner – The session id of the owner of this znode if the znode is an ephemeral node. If it is not an ephemeral node, it will be zero.
* dataLength – The length of the data field of this znode.
* numChildren – The number of children of this znode.

   **Watches监测**

ZooKeeper中的Watch是只能触发一次。也就是说，如果客户端在指定的ZNode设置了Watch，如果该ZNode数据发生变更，ZooKeeper会发送一个变更通知给客户端，同时触发设置的Watch事件。如果ZNode数据又发生了变更，客户端在收到第一次通知后没有重新设置该ZNode的Watch，则ZooKeeper就不会发送一个变更通知给客户端。

 ZooKeeper异步通知设置Watch的客户端。但是ZooKeeper能够保证在ZNode的变更生效之后才会异步地通知客户端，然后客户端才能够看到ZNode的数据变更。由于网络延迟，多个客户端可能会在不同的时间看到ZNode数据的变更，但是看到变更的顺序是能够保证有序一致的。

 ZNode可以设置两类Watch，一个是Data Watches（该ZNode的数据变更导致触发Watch事件），另一个是Child Watches（该ZNode的孩子节点发生变更导致触发Watch事件）。调用getData\(\)和exists\(\) 方法可以设置Data Watches，调用getChildren\(\)方法可以设置Child Watches。调用setData\(\)方法触发在该ZNode的注册的Data Watches。调用create\(\)方法创建一个ZNode，将触发该ZNode的Data Watches；调用create\(\)方法创建ZNode的孩子节点，则触发ZNode的Child Watches。调用delete\(\)方法删除ZNode，则同时触发Data Watches和Child Watches，如果该被删除的ZNode还有父节点，则父节点触发一个Child Watches。

 另外，如果客户端与ZooKeeper Server断开连接，客户端就无法触发Watches，除非再次与ZooKeeper Server建立连接。 



**Sequence Nodes（序列节点）** 

在创建ZNode的时候，可以请求ZooKeeper生成序列，以路径名为前缀，计数器紧接在路径名后面，例如，会生成类似如下形式序列：

| `1` | `qn-0000000001, qn-0000000002, qn-0000000003, qn-0000000004, qn-0000000005, qn-0000000006, qn-0000000007` |
| --- | --- |

对于ZNode的父节点来说，序列中的每个计数器字符串都是唯一的，最大值为2147483647。

**ACLs（访问控制列表）** 

ACL可以控制访问ZooKeeper的节点，只能应用于特定的ZNode上，而不能应用于该ZNode的所有孩子节点上。它主要有如下五种权限：

* CREATE 允许创建Child Nodes
* READ 允许获取ZNode的数据，以及该节点的孩子列表
* WRITE 可以修改ZNode的数据
* DELETE 可以删除一个孩子节点
* ADMIN 可以设置权限

ZooKeeper内置了4种方式实现ACL：

* world 一个单独的ID，表示任何人都可以访问
* auth 不使用ID，只有认证的用户可以访问
* digest 使用username:password生成MD5哈希值作为认证ID
* ip 使用客户端主机IP地址来进行认证

 

# Zookeeper服务注册与发现：

## 服务化背景

多数公司的软件开发都是从一个单一的系统开始，以最快的速度进行迭代开发，而随着公司业务的发展，在原有系统上的功能添加导致系统越来越庞大，其带来了几个问题：

1. 单一系统部署在单机上，随着访问量的提升，单机的性能受限，降低了服务的可用性。
2. 系统的维护复杂，启动速度非常缓慢，日志复杂等
3. 同时单一系统导致了一个复杂系统都必须由同一种语言进行开发

如何解决这些问题，业务服务化是个有效的手段来解决大规模系统的性能瓶颈和复杂性，通过拆分，可以将系统拆分成不通的子系统，其带来的好处是：
1. 系统压力得到分流，同时方面扩展，对压力较大的子系统进行相关扩展。
2. 各个业务模块的业务逻辑少，系统维护性提高

同时服务化也带来了一系列的问题：
1. 随着系统服务越来越多，如何管理这些服务？

1. 如何把请求发送到提供统一服务的不通主机上？
2. 如果提供服务的主机有上下线等，如何将这些信息提供给依赖方？

zookeeper是解决上述问题的一种方案，zookeeper官方描述，其能够：

1. 配置信息的存储中心服务器
2. 命名服务
3. 分布式的协调（锁）
4. Master选举

Zookeeper作为服务注册和发现的解决方案，其优点如下：
1. 简单api，
2. 多个公司选择，阿里（Dubbo）以及国外多个公司的选择
3. 支持多种语言客户端

![](/assets/wKioL1dF09nw54OhAAA-GqTDZPI375.jpg-s_3885195311.jpg)

## 服务提供者

服务提供者作为服务的提供方将自身的服务信息注册到Zk（注册中心），服务信息包括：

1. 隶属哪个系统
2. 服务的ip和端口
3. 服务请求的url
4. 服务的权重等

## 服务注册中心

服务注册中心主要提供所有的服务注册信息的中心存储，同时将服务注册信息的更新通知给服务消费者。

## 服务消费者

服务消费者主要职责：

1. 服务消费者启动时从服务注册中心获取需要的服务注册信息
2. 将服务注册信息缓存在本地
3. 监听服务注册信息的变更，如接手到服务注册中心的变更通知，更新本地缓存
4. 根据本地缓存中的服务注册信息构建服务请求调用列表，并根据负载均衡等策略进行请求分发
5. 对服务提供方的存活进行检测，如果出现不可用的服务提供方，将从本地缓存中剔除。

举例如下
![](/assets/zk.png)

在zk的根目录下形成了一颗树\/Company\/Services\/Porvider2\/service2\/producers\/URI,其中company只定为公司或者大部门，services表示这个是微服务属性，Provider表示服务提供方，一个服务提供方可能提供多个服务，service为服务的id，producers为服务的生产者，考虑到负载均衡，高可用等，一个生产者可能部署在多个节点上或者多个进程，提供多个URI进行访问。consumers表示一个服务被哪些消费者进行消费，对服务的管理起到重要作用。



# zk分布式索







