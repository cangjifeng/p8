

zookeeper 入门指南

```
zookeeper 官网地址
https://zookeeper.apache.org/

zookeeper 入门学习指南
https://zookeeper.apache.org/doc/current/index.html
```



了解zookeeper

```
zookeeper 是一个分布式协调服务，它是用来协同别人工作的。

zookeeper 是复制结构（主从模型），意味着有主从节点，主节点叫leader节点，从节点叫follower节点，数据写入在leader节点然后，leader节点往follower 节点同步数据，可以从leader节点和follower节点读取数据。

zookeeper具有高可用、可靠性的特点。

ZK是基于CAP的落地方案之一，优先选择P 分区容性，正在C 一致性和A 可用性中选择了 C ，放弃了A ，意味着放弃了可用性，但是zk的恢复是非常快的，官网给出的是200ms能完成选举leader工作（不可用状态到可用状态的切换是非常快的，paxos 算法实现 ZAB选主），所以zk也是具备高可用的。
```



![](D:\java-code\cangjifeng\p8\zookeeper\zookeeper-images\zkservice.jpg)



zookeeper 集群中 首先要有如下几个认知

```
1 leader 肯定会挂
2 leader 挂了，服务不可用
3 服务不可用，认为它是一个不可靠的集群
4 事实，zookeeper 是及其可靠和高可用的
5 如果leader 挂了，需要一种方式可用迅速地恢复出一个leader
```



session的概念

```
每个客户端 client 连接到服务端 server，都会有一个session。叫一个会话，session 有创建到消亡的声明周期。
```



zookeeper 的2种运行状态

```
1 可用状态，leader 挂了，也叫无主模式
2 并不用状态，leader 正常工作，也叫有主模式
3 不可用状态恢复到可用状态越快越好，官网给出的恢复一个leader在200ms内
```

使用的认知

```
redis 也是分布式的，但其可以作为数据库使用，

但zookeeper 不要当做数据库使用
```



zookeeper 的结构

```
zookeeper 是一个目录结构，每个节点可以存储1MB数据，节点分为2种，一种是临时节点，一种是永久节点。
临时节点是关联到session上的。临时节点和永久节点支持序列化。

zookeeper 的节点是存在在内存中的，同时支持持久化。
```

zookeeper提供的一些特征保证

```
ZooKeeper非常快速且非常简单。但是，由于其目标是作为构建更复杂的服务（例如同步）的基础，因此它提供了一组保证。这些是：

1 顺序一致性-来自客户端的更新将按照发送的顺序应用。
2 原子性-更新成功或失败。没有部分结果。
3 单个系统映像-无论客户端连接到哪个服务器，客户端都将看到相同的服务视图。也就是说，即使客户端故障转移到具有相同会话的其他服务器，客户端也永远不会看到系统的较旧视图（因为zookeeper 是一个复制模型）。
4 可靠性-应用更新后，此更新将一直持续到客户端覆盖更新为止。
5 及时性-保证系统的客户视图在一定时间内是最新的。
```

zookeeper 支持原语操作

```
ZooKeeper的设计目标之一是提供一个非常简单的编程界面。因此，它仅支持以下操作：

create：在树中的某个位置创建一个节点
delete：删除节点
exists ：测试某个位置是否存在节点
get data：从节点读取数据
set data：将数据写入节点
get children：获取节点子节点的列表
sync：等待数据传播
```



zookeeper 的几个重点

```
1 zookeeper的原理
2 paxos leader选取算法
3 ZAB 原子广播协议
4 API，不怕写zk的客户端
5 callback  ->reactive 模型， 响应式编程 (更充分压寨 OS HW 性能)
```



zookeeper 的原理

```
zk是一个协调器，具有可扩展性，时序性，可靠性，快速。

可扩展性：
zk是用于框架架构，有2个角色 leader 和follower，还有一个 observer （监视器）。只有follower才能选举。选举的速度和效率有follower的数量决定。
zk的可扩展性还表现在读写分离。
zoo.cfg
server.1=node01:2888:3888
server.2=node02:2888:3888
server.3=node03:2888:3888
server.4=node04:2888:3888:observer

可靠性：
zookeeper，核心设计思想：攘其外必先安其内（不可用时候，leader 挂了时候，能够快速恢复）
快速（leader选举快，一定有主有leader，快速恢复leader）
数据的可靠和可用和一致性，对外提供服务（攘其外），一致性，最终一致性，更倾向于强一致性（过程中是否对外提供服务）。分布式环境下。

百度搜索技巧，paxos site:douban.com
```



paxos

```

Paxos，它是一个基于消息传递的一致性算法，Leslie Lamport在1990年提出，近几年被广泛应用于分布式计算中，Google的Chubby，Apache的Zookeeper都是基于它的理论来实现的，Paxos还被认为是到目前为止唯一的分布式一致性算法，其它的算法都是Paxos的改进或简化。有个问题要提一下，Paxos有一个前提：没有拜占庭将军问题。就是说Paxos只有在一个可信的计算环境中才能成立，这个环境是不会被入侵所破坏的。

Paxos描述了这样一个场景，有一个叫做Paxos的小岛(Island)上面住了一批居民，岛上面所有的事情由一些特殊的人决定，他们叫做议员(Senator)。议员的总数(Senator Count)是确定的，不能更改。岛上每次环境事务的变更都需要通过一个提议(Proposal)，每个提议都有一个编号(PID)，这个编号是一直增长的，不能倒退。每个提议都需要超过半数((Senator Count)/2 +1)的议员同意才能生效。每个议员只会同意大于当前编号的提议，包括已生效的和未生效的。如果议员收到小于等于当前编号的提议，他会拒绝，并告知对方：你的提议已经有人提过了。这里的当前编号是每个议员在自己记事本上面记录的编号，他不断更新这个编号。整个议会不能保证所有议员记事本上的编号总是相同的。现在议会有一个目标：保证所有的议员对于提议都能达成一致的看法。

好，现在议会开始运作，所有议员一开始记事本上面记录的编号都是0。有一个议员发了一个提议：将电费设定为1元/度。他首先看了一下记事本，嗯，当前提议编号是0，那么我的这个提议的编号就是1，于是他给所有议员发消息：1号提议，设定电费1元/度。其他议员收到消息以后查了一下记事本，哦，当前提议编号是0，这个提议可接受，于是他记录下这个提议并回复：我接受你的1号提议，同时他在记事本上记录：当前提议编号为1。发起提议的议员收到了超过半数的回复，立即给所有人发通知：1号提议生效！收到的议员会修改他的记事本，将1好提议由记录改成正式的法令，当有人问他电费为多少时，他会查看法令并告诉对方：1元/度。

现在看冲突的解决：假设总共有三个议员S1-S3，S1和S2同时发起了一个提议:1号提议，设定电费。S1想设为1元/度, S2想设为2元/度。结果S3先收到了S1的提议，于是他做了和前面同样的操作。紧接着他又收到了S2的提议，结果他一查记事本，咦，这个提议的编号小于等于我的当前编号1，于是他拒绝了这个提议：对不起，这个提议先前提过了。于是S2的提议被拒绝，S1正式发布了提议: 1号提议生效。S2向S1或者S3打听并更新了1号法令的内容，然后他可以选择继续发起2号提议。

好，我觉得Paxos的精华就这么多内容。现在让我们来对号入座，看看在ZK Server里面Paxos是如何得以贯彻实施的。

小岛(Island)——ZK Server Cluster

议员(Senator)——ZK Server

提议(Proposal)——ZNode Change(Create/Delete/SetData…)

提议编号(PID)——Zxid(ZooKeeper Transaction Id)

正式法令——所有ZNode及其数据

貌似关键的概念都能一一对应上，但是等一下，Paxos岛上的议员应该是人人平等的吧，而ZK Server好像有一个Leader的概念。没错，其实Leader的概念也应该属于Paxos范畴的。如果议员人人平等，在某种情况下会由于提议的冲突而产生一个“活锁”（所谓活锁我的理解是大家都没有死，都在动，但是一直解决不了冲突问题）。Paxos的作者Lamport在他的文章”The Part-Time Parliament“中阐述了这个问题并给出了解决方案——在所有议员中设立一个总统，只有总统有权发出提议，如果议员有自己的提议，必须发给总统并由总统来提出。好，我们又多了一个角色：总统。

总统——ZK Server Leader

leader 选举过程 ……

现在我们假设总统已经选好了，下面看看ZK Server是怎么实施的。

情况一：

平民甲(Client)到某个议员(ZK Server)那里询问(Get)某条法令的情况(ZNode的数据)，议员毫不犹豫的拿出他的记事本(local storage)，查阅法令并告诉他结果，同时声明：我的数据不一定是最新的。你想要最新的数据？没问题，等着，等我找总统Sync一下再告诉你。

情况二：

平民乙(Client)到某个议员(ZK Server)那里要求政府归还欠他的一万元钱，议员让他在办公室等着，自己将问题反映给了总统，总统询问所有议员的意见，多数议员表示欠平民的钱一定要还，于是总统发表声明，从国库中拿出一万元还债，国库总资产由100万变成99万。平民乙拿到钱回去了(Client函数返回)。

情况三：

总统突然挂了，议员接二连三的发现联系不上总统，于是各自发表声明，推选新的总统，总统大选期间政府停业，拒绝平民的请求。



参考： https://www.douban.com/note/208430424/
```



ZAB有主

```

ZAB 是zookeeper的原子广播协议，作用在可用状态下，有leader 。
ZAB协议中的原子，指的是原子性，要么成功，要么失败，不能有中间状态；
ZAB协议中的广播，指的是分布式多节点通知，通知并不意味着所有节点都收到，只要过半即可；

流程如下：
1 client客户端发起写操作，发送给其中一个follower节点；
2 收到客户端请求的follower节点把请求转发给leader节点；
3 leader节点收到follower节点的写操作，会生成一个事务id，叫ZXID（唯一的）；
4 因为zk的数据状态是存储在内存中的，并用磁盘保存日志，leader节点维护一个通知队列（为每个节点之前都维护这一个队列），把写操作的结果通知给follower节点（期望是所有的follower节点都能收到该通知）
4-1 leader 发送给follower，[是2PC的第一阶段]follower节点收到通知执行写磁盘；
4-1-x follower 给leader 回复，结果可能是OK（回复很快速），但不是所有节点都能回复（可能有网络延迟或者网络抖动）
4-2 （当leader知道已经有过半回复），leader会发起第二阶段[是2PC的第二阶段]，让每个follower执行写操作，指的是把数据写入内存；
5 leader 最后把收到的请求（第2步骤）回复给follower，发起请求的follower再返回给客户端client。
此时其他客户端可以读取操作，读到leader节点或者2步骤中的follower节点一定是最新数据，但读到有些节点（可能没有收到leader的通知，即没有同步到leader的最新数据）可能不是最新数据，有一个可选的机制实现leader节点的数据和follower之间数据同步（sync）。

```



ZAB无主

```

leader选取分2种情况选举：
1 第一次启动集群，选取leader
2 重启集群，leader挂了后，重新选取leader

每个节点node会存2个编号
1 一个编号是标识节点的myid
2 另一个编号是zxid 事务id

什么样的节点可以作为leader节点
经验最丰富的zxid，当zxid都一样，再看myid。所有的数据都来自过半通过


zk选取过程
1 :3888端口号造成两两通信
2 只要任何人投票，都会触发那个准leader发起自己的投票
3 推选制，先比较zxid，zxid相同在比较myid
```



watch

```
watch 监控 观察
统一视图
目录结构树
zk是协调器，协调别人工作的，zk只起到协调作用。客户端会getnode节点信息（指的是结构树），并watchnode节点信息。
客户端之间是有通信的，客户端可以自己做心跳去监控zk。
当客户端挂掉，该客户端和zk的session会断掉，zk的表现是目录结构树发生变化（某个节点被删除，ms级别的），当某个节点被删除会产生一个时间event（create、delete、change、children等事件），时间发生会主动的回调观察者节点client

```



zookeeper API

```
zookeeper的maven依赖

<!-- https://mvnrepository.com/artifact/org.apache.zookeeper/zookeeper -->
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.4.6</version>
</dependency>

重点关注：
1 zookeeper是基于session连接的，没有连接池的概念
2 关注 watch zk 等接口和类的使用，做create、delete、change、children、get等操作
```



基于zookeeper的分布式做的实现

```
基于zk的分布式锁的实现，建议是使用临时节点，并加过去时间
```

