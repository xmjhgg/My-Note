# Zookeeper总结

什么是Zookeeper：

Zookeeper是一个分部署服务框架，主要可以用来解决分布式应用中数据一致性的问题，比如：统一命名服务，分布式配置管理，分布式锁，分部署协调等

简单来说就是一个 **文件系统+监听通知**



[TOC]



## Zookeeper架构

![](E:\documentary\mdImgs\Zookeeper\1.jpg)

Zookeeper的架构如上图



### 角色功能说明

- Leader

  Leader是Zookeeper的核心角色，所有的事务请求（写操作）都需要转发给Leader进行统一处理，并会为每个事物请求进行顺序编号（ZXID）

- Follower

  处理读操作，并将写操作请求转发给Leader，参与Leader的选举，还可以针对读请求较多的Zookeeper集群增加Obsever角色

- Observer

  和Follower相似，但Observer只处理读操作转发写操作，不参与Leader的选举



## Zookeeper数据节点

### Zookeeper的存储结构图

![](E:\documentary\mdImgs\Zookeeper\2.jpg)

上图就是Zookeeper的存储结构，每个节点以"/"分割，根节点就是一个空字符串的节点



### 节点数据详细说明

Zookeeper的存储结构是一个树形的结构，树的节点称为Znode，节点以key-value保存数据，key就是节点的名字，value则是我们自定义的数据和Zookeeper自带的数据，每个节点默认最大存储1MB的数据

- State：此为状态信息，描述该Znode版本、权限等信息。
- Data：用户自定义的数据
- Children：该Znode下的子节点

- Version：每个znode都有版本号，这意味着每当与znode相关联的数据发生变化时，其对应的版本号也会增加。当多个zookeeper客户端尝试在同一znode上执行操作时，版本号的使用就很重要。
- ACL  -  ACL基本上是访问znode的认证机制。它管理znode读取和写入操作的权限。
- Date  - 时间戳表示创建和修改znode所经过的时间。它通常以毫秒为单位。ZooKeeper从“事务ID"(zxid)标识znode的每个更改。**Zxid** 是唯一的，并且为每个事务保留时间，以便可以轻松地确定从一个请求到另一个请求所经过的时间。
- Length：存储在znode中的数据总量是数据长度。最多可以存储1MB的数据。



### 节点类型

Znode的类型分为持久节点、临时节点、顺序节点

- 持久节点：持久节点是指当客户端和Server断开后，Server仍会保存这个节点，默认创建的节点都是持久节点
- 临时节点：临时节点当客户端和Server断开后，这个节点会被删除。临时节点不允许有子节点。
- 顺序节点：顺序节点可以使持久节点也可以是顺序节点，当一个新的Znode被创建或修改为顺序节点时，ZooKeeper通过将10位的序列号附加到原始名称来设置znode的路径。例如，如果将具有路径 /myapp 的znode创建为顺序节点，则ZooKeeper会将路径更改为 /myapp0000000001 ，并将下一个序列号设置为0000000002。利用这个特性可以实现Zookeper的同步



## Zookeeper数据一致性机制

Zookeeper集群的数据一致性机制的本质就是所有的Server都执行相同的修改操作

如果所有的Server在同步过一次Leader的数据后，后续只要和Leader执行同样的修改操作，那么集群中的所有Server的数据就是一致的

### 写操作执行流程

![](E:\documentary\mdImgs\Zookeeper\3.jpg)

1. Client向Zookeeper发送一个写请求
2. Server接收到请求后转发给Leader，Leader会将这个请求转发给所有的Server，各个Server将这个请求加入到自己的待写队列中，并给Leader返回确认信息
3. 当Leader收到半数以上的Server的成功信息，那么Leader就会向各个Server发送提交信息，各个Server就会执行队列中的写操作
4. 执行完写操作后，Server就会返回给Client执行成功的信息



## Zookeeper的选举机制

在Zookeeper中，Leader是通过选举机制产生，当一个Zookeeper集群新启动或者当Leader宕机时，都会触发Zookeeper的选举机制

Leader的当选条件是获得超过节点个数一半的选票，如3个节点，则得到2票即当选Leader。



### 选举机制前置说明

#### Server的选举状态

- Looking：说明当前的Server处于正在寻找Leader的状态
- Following：说明当前的Server是Follower
- Observering：说明当前的Server是Observer
- Leading：说明当前的Server是Leader

#### 事务ID(zxid)

Zookeeper中，每当Zookeeper的数据状态发生变化都会生成一个唯一的zxid，由Leader统一分配，不断递增的64位数。zxid表明了所有的ZooKeeper 的数据变更顺序，如果 zxid1 小于 zxid2 说明 zxid1 在 zxid2 之前发生。



下面分两种情况来说明：

### 集群初始启动时的选举

![](E:\documentary\mdImgs\Zookeeper\8.jpg)

Zookeeper集群刚启动时，所有的节点的数据状态都相同，此时每个节点的zxid都相同，所以这里选举主要依靠myid（*myid可以在配置文件中配置*）



假设每台服务器都是按照顺序启动，那么选举就会如下进行：

1. 当第一台服务器启动时，其无法单独完成选举机制，这台服务器还是Looking状态
2. 当第二台服务器启动后，两台服务器可以互相通信，

3. Sever1 和 Server2 都会将自己作为 Leader 服务器来进行投票，每次投票会包含所推举的服务器的 myid 和 ZXID，使用(myid,  zxid)来表示，此时 Sever1 的投票为(1, 0)，Server2 的投票为(2, 0)，然后各自将这个投票发给集群中其他机器。　

4. Sever1和Server2接受来自各个服务器的投票。集群的每个服务器收到投票后，首先判断该投票的有效性，如检查是否是本轮投票、是否来自 LOOKING 状态的服务器。　

5. 处理投票

   Server1接收到Server2的投票，首先会比较Server2的投票和自己刚刚投的票，先比较zxid，两者相等，再比较myid，Server2比较大，所以Server1会更新Server2，并将票投给Server1

6. 统计选票

   此时Server2有两票，但是Server2的票数还没有到达总节点个数的一半，此次的选举无效

7. 第三台服务器启动，集群还没有Leader产生，再进行一次选举，按照上面的流程，Server3将会收到3张选票，成为Leader状态更新为Leading，Server1和Server2更新为Following

8. 第四台第五台服务器启动时，集群中已经有Leader，Server4和Server5将直接从Looking状态更新为Following

### Leader宕机时的选举

在 Zookeeper 运行期间，如果 Leader 节点挂了，那么整个 Zookeeper 集群将先暂停对外服务，进入新一轮Leader选举

假设只有三台服务器Server123，刚刚票选的Leader Server3挂掉了

1. 变更状态。Leader 挂后，余下的非 Observer 服务器都会讲自己的服务器状态变更为 LOOKING，然后开始进入 Leader 选举过程。　　

2. 每个Server会发出一个投票。在运行期间，每个服务器上的 ZXID 可能不同，此时假定 Server1的 ZXID 为  124，Server2的 ZXID 为 123；在第一轮投票中，Server1和 Server2都会投自己，产生投票(1, 124)，(3,  123)，然后各自将投票发送给集群中所有机器。　　

3. 接收来自各个服务器的投票。与启动时过程相同。　　

4. 处理投票。与启动时过程相同，由于 ZK1 事务 ID 大，ZK1 将会成为 Leader。　　

5. 统计投票。与启动时过程相同。　　

6. 改变服务器的状态。与启动时过程相同。



## Zookeeper的使用案例

### 分布式统一配置项管理

![](E:\documentary\mdImgs\Zookeeper\5.jpg)

使用场景：

1. 在分布式系统中，所有的节点配置项应该是一致的
2. 当配置项修改后，能尽快的同步到每个节点上

使用方法：

每个客户端监听配置项的Znode，当这个Znode修改后，Zookeeper会通知所有的Client

### 统一命名服务

![](E:\documentary\mdImgs\Zookeeper\6.jpg)

使用场景：

在分布式系统中，某些应用要通过某个标识去访问另一个服务器时，可以统一命令，忽略IP地址等独特信息

使用方法：

当需要通过某个标识去访问IP时，就可以将地址设置在这个标识的子节点下，而不用特意去设置IP地址

### 统一集群节点管理

![](E:\documentary\mdImgs\Zookeeper\7.jpg)

使用场景：分布式系统中，需要实时的掌控集群中每个节点的运行情况

使用方法：集群中的每个节点都向Zookeeper创建一个独属的节点，定期将自身的状况更新到这个节点上