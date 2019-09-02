# Zookeeper

[官网](https://zookeeper.apache.org/)

部署模式：

* 单机
* 集群，但是我个人认为，zookeeper都是集群的存在才有意义

## Zookeeper常用的shell命令

[群起zookeeper的脚本地址](<https://blog.csdn.net/qq_31807385/article/details/84975964>)

~~~shell
# 启动zookeeper服务器：
[root@M1 /home/pyd/zookeeper-3.4.10]# ./bin/zkServer.sh start

# 关闭zookeeper服务器：
[root@M1 /home/pyd/zookeeper-3.4.10]# ./bin/zkServer.sh stop

# 查看状态：
[root@M1 /home/pyd/zookeeper-3.4.10]# ./bin/zkServer.sh status
Mode:standalone  # 这里表示的是standalone模式，单节点
 
# 启动zookeeper的客户端
[root@M1 /home/pyd/zookeeper-3.4.10]# ./bin/zkCli.sh

###############在zookeeper中的相关命令##################
# 查看znode中包含的节点，ls2表示的是获取节点的详细信息
[zk: localhost:2181(CONNECTED) 3] ls /
[zookeeper]

# 获取到节点的值
[zk: localhost:2181(CONNECTED) 4] get /zookeeper

cZxid = 0x0
ctime = Thu Jan 01 08:00:00 CST 1970
mZxid = 0x0
mtime = Thu Jan 01 08:00:00 CST 1970
pZxid = 0x0
cversion = -1
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 0
numChildren = 1 # 表示当前的节点是/ 有一个子节点
~~~

### 临时节点

zookeeper中的节点有两种类型：

* 持久节点：客户端和服务器端断开连接后，创建的节点不删除
* 短暂节点：客户端和服务器断开之后，创建的节点自己删除 `create -e`

~~~shell
[zk: localhost:2181(CONNECTED) 6] create /number "数字"   # 创建数字
Created /number
[zk: localhost:2181(CONNECTED) 7] ls /     # 可以看见数据的的变化           
[number, zookeeper]

# 创建一个临时的节点，并查看
[zk: localhost:2181(CONNECTED) 8] create -e /number/one "1"
Created /number/one
[zk: localhost:2181(CONNECTED) 9] get /number/one
1
cZxid = 0x100000007
ctime = Sun Sep 01 09:01:21 CST 2019
mZxid = 0x100000007
mtime = Sun Sep 01 09:01:21 CST 2019
pZxid = 0x100000007
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x6cea44f22b0001
dataLength = 1
numChildren = 0  # 可以查看当前的节点的子节点

# 退出之后在登录，ls /number 没有结果
[zk: localhost:2181(CONNECTED) 0] ls /number
[]
~~~

### 监听节点变化

这里演示，监听节点路径的变化，监听节点的值的变化

~~~shell
# 监听节点值的变化，使用get
[zk: localhost:2181(CONNECTED) 1] get /number watch
数字
cZxid = 0x100000006
ctime = Sun Sep 01 08:59:41 CST 2019
mZxid = 0x100000006
mtime = Sun Sep 01 08:59:41 CST 2019
pZxid = 0x100000008
cversion = 2
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 6
numChildren = 0
[zk: localhost:2181(CONNECTED) 2] 
WATCHER::

WatchedEvent state:SyncConnected type:NodeDataChanged path:/number  # 该部分是下面的代码执行了之后的结果

# 修改值
[zk: localhost:2181(CONNECTED) 6] set /number "数字大爆发"


# 监听路径的变化
[zk: localhost:2181(CONNECTED) 3] ls /number watch
[]
[zk: localhost:2181(CONNECTED) 4]   # 该部分是下面的代码执行了之后的结果
WATCHER::

WatchedEvent state:SyncConnected type:NodeChildrenChanged path:/number

# 增加路径
[zk: localhost:2181(CONNECTED) 8] create /number/one "one"
Created /number/one
~~~



## 分布式系统一致性

### NameNode的高可用实现

#### 问题1：数据同步

如果HDFS要实现高可用的话，需要两 个namenode一个active，一个standby ，standby需要有active中的元数据，否则在active宕机的时候无法完成替换的效果。 所以两台namenode面临的第一个问题就是数据的一致性问题 即元数据的同步，active定时给standby拷数据，假设active宕机，standby中的数据是不全的。 

#### 解决方案：

弄一个第三方，这个第三方记录active 的edits文件，active的fsimage文件不能写入这个第三方，因为这会导致大量的io，但是standby需要active的fsimage文件，解决的办法是，namenode在第一次格式化之后，同步到standby。 active向第三方进行写操作，standby对这个第三方读 如此一来，我们实现了数据的同步，所以，standby和active的唯一区别是edit_process文件的有无。

#### 问题2：数据一致（脑裂）

**HDFS的NameNode的脑裂**：HDFS所有的元数据信息都存储在NameNod上，为了保证整个集群的高可用，一般会部署两个NameNode，一个NameNode做为Active，另外一个NameNode作为Standby，如果应用（Clint）或者是DataNode对于主服务器是否可用的判断不同，那么就会导致HDFS集群的混乱，例如两个应用要对HDFS进行写操作，一个认为是A可用，就将所有的信息写到的A-NameNode，另外一个认为A不可用，B可用于是将数据写到了B-NameNode，于是两个应用拿到了同样的文件路径，并且获得了写权限，于是同一个路径指向了两个不同的文件，类似这样的问题，在分布式系统中称之为**脑裂**。

#### 解决方案

为了保证数据的一致性，防止脑裂的产生，可以按照下面的方式来避免脑裂的出现，如果一个上位，直接把原来的active干掉：

![](img/zk/1.png)

基于以上的架构，我们能够实现NameNode的手动故障转移 ，如何实现故障之后的自动切换？此时，我们需要一个裁判来判断，哪个服务器是主服务器，同时这个裁判也必须要高可用，所以这个裁判应该是多个服务器构成的”裁判团“，那么问题又来了，裁判怎么防止脑裂呢？此时需要一个**多台服务器状态一致**的解决方案，比较常用的就是**Zookeeper**。

![](img/zk/2.png)



这里的zkfc是zookeeper集群的客户端

*  获取namenode的状态，
* namenode的状态；

active负责在zookeeper中注册锁，这个锁其实就是一个文件夹，然后另外一个zkfc （standby的）来监听这把锁，一旦active 发生了变化，会通过删除锁的方法来告知standby，standby 就能够监听到这个变化，会将standby变为active。

## zookeeper架构：

zookeeper可以说是一个提供锁服务的分布式系统，它由多台服务器构成一个集群对外提供锁服务，应用程序连接到任意一台服务器都可以获取或者是释放锁，因此这些服务器必须保持严格的状态一致，不能出现一台服务器将锁交给一台服务器，另外一台服务器将锁交给另外一台服务器。如何保证一把锁只是交给一个应用？Paxos就是用来解决这个问题的。如下：

![](img/zk/3.png)

应用服务器连接到任意一台服务器后提起状态修改申请，从图中看也就是服务器1，会将这个请求发给集群中的其他服务器进行表决，表决结果会发送给其他所有服务器，如果某个服务器同时收到了另一个应用程序同样的修改请求，它可能会拒绝服务器 1 的表决，并且自己也发起一个同样的表决请求，那么其他服务器就会根据时间戳和服务器排序规则进行表决。

表决结果会发送给其他所有服务器，最终发起表决的服务器也就是服务器 1，会根据收到的表决结果决定该修改请求是否可以执行，从而在收到请求的时候就保证了数据的一致性。zookeeper中使用paxos算法的简化版；**ZAB协议（原子消息广播协议）**来保证zookeeper中数据的一致性。

ZooKeeper 系统中所有服务器都存储相同的数据，并且每次数据的更新都需要所有的服务器表决，所以和一般的分布式系统相反，性能会随着数量的增加而下降。

![](img/zk/5.png)

Zookeeper全局的数据是一致的，其数据结构和类Unix的文件系统很类似，每个路径下面可以存储数据，默认存储的大小是1M：zookeeper通过树状结构的方式来记录数据：

![](img/zk/4.png)

大数据集群通常都是主从架构，主服务来管理集群的状态和元数据的信息，为了防止脑裂，在运行期间只能有一个服务工作，同时为了高正高可用，必须有另外一台服务器保持热备，那么应用服务和集群中的其他服务器如何才能知道当前的那个服务器是工作的主服务器呢？所以很多的大数据系统都依赖Zookeeper来提供一致性数据服务，用于选举主服务器。一台主服务器启动之后向zookeeper注册自己为主服务器，后来向zookeeper注册的就只能是热备服务器，应用程序运行期间和主服务器通信。

如果当前的主服务器宕机（zookeeper上注册的心跳数据不在更新）热备服务器通过zookeeper的监控机制发现当前工作的主服务器宕机，就想zookeeper注册自己为当前工作的主服务器，应用和集群中的其他服务器会和新的主服务器进行通信。

## Zookeeper的工作机制

Zookeeper的设计基于观察者模式，zookeeper集群负责存储大家都关系的数据，一旦这些数据发生了变化，zookeeper就负责通知已经在zookeeper上注册的观察者。

![](img/zk/6.png)

**小知识**：

* Quorum：是仲裁的意思，启动zookeeper之后会有一个**QuorumPeerMain**进程。
* Zookeeper的配置文件zoo.conf 中的两个选项：
* dataDir，是数据文件的目录+树持久化路径，主要有用于保存zookeeper中的数据。
* clientPort：服务器监听客户端的端口号。
* 2888就是Leader和Follower信息交换的端口；3888 是在Leader挂了之后，用来选举的端口。
* 2就是myid的值，myid表示的是zookeeper的编号，放置在`dataDir`所在的目录下
* 常见的命令：

~~~shell
create /isea/gril “beauty” # 创建节点，并添加值：
get /isea/gril # 获取节点的值：
delte   # 删除节点
rmr    # 递归删除节点
~~~

集群模式下，zoo.conf 中的

![](img/zk/7.png)

## zookeeper的内部原理

### zookeeper的节点类型

* 持久（Persistent）客户端和服务器建立连接值断开后，创建的节点不删除

* 短暂（Ephemeral）客户端和服务器断开连接之后，创建的节点自己删除

### 监听器原理

![](img/zk/8.png)

* 首先有一个主线程，在主线程中创建zkClient，该客户端会有两个线程，listenner和connect
* connect线程会将注册到的监听事件发送给zookeeper
* 在zookeeper中的监听器会将connect发送过来的事件注册到监听列表中
* zookeeper发现注册的事件有变化的时候，会将这个消息发送给listenner
* listenner线程调用zkcli的process方法进行相应的处理

### 选举机制

* 半数机制，集群中半数以上的机器存活，集群可用，所以zookeeper适合安装**奇数**台机器
* Zookeeper在配置文件中没有指定Master和Slave，zookeeper中的Leader是通过内部选举机制完成的。

* 选举机制：

![](img/zk/9.png)

* 服务器1启动，发起一次选举。服务器1投自己一票。此时服务器1票数一票，不够半数以上（3票），选举无法完成，服务器1状态保持为LOOKING；

* 服务器2启动，再发起一次选举。服务器1和2分别投自己一票并交换选票信息：此时服务器1发现服务器2的ID比自己目前投票推举的（服务器1）大，更改选票为推举服务器2。此时服务器1票数0票，服务器2票数2票，没有半数以上结果，选举无法完成，服务器1，2状态保持LOOKING

* 服务器3启动，发起一次选举。此时服务器1和2都会更改选票为服务器3。此次投票结果：服务器1为0票，服务器2为0票，服务器3为3票。此时服务器3的票数已经超过半数，服务器3当选Leader。服务器1，2更改状态为FOLLOWING，服务器3更改状态为LEADING；

* 服务器4启动，发起一次选举。此时服务器1，2，3已经不是LOOKING状态，不会更改选票信息。交换选票信息结果：服务器3为3票，服务器4为1票。此时服务器4服从多数，更改选票信息为服务器3，并更改状态为FOLLOWING；

* 服务器5启动，同4一样当小弟。

### 写请求的实现

![](img/zk/10.png) 







