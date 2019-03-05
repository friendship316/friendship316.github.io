---
title: zookeeper初印象
date: 2018-05-27 17:44:24
---
炒鸡简单的zookeeper初印象总结。单机安装zookeeper集群请参考[zookeeper安装与伪集群配置](https://www.jianshu.com/p/f13a17e3e351)。

### 数据结构
#### znode基本特性
zookeeper服务会维护一个具有层次关系的数据结构，类似于Linux文件系统。其中每一个节点称为znode，如下图所示：

![zookeeper数据结构](https://upload-images.jianshu.io/upload_images/3727888-de9f9872f7913ad4.png)

每个znode包含三部分信息：
>1. stat. 此为状态信息, 描述该znode的版本, 权限等信息；
>2. data. 与该znode关联的数据；
>3. children. 该znode下的子节点；

znode具有如下特性：

>1. 每个znode的唯一标识是路径；
>2. 拥有两种类型：永久节点(persistent)和临时节点(ephemeral)；
临时节点：一旦创建这个 znode 的客户端与服务器失去联系，这个 znode 也将自动删除，Zookeeper 的客户端和服务器通信采用长连接方式，每个客户端和服务器通过心跳来保持连接，这个连接状态称为 session，如果 znode 是临时节点，这个 session 失效，znode 也就删除了。**临时节点不能拥有子节点**；
>3. znode具有版本，也就是说一个节点可以存储多份数据；
>4. znode 的目录名可以自动编号，如 App1 已经存在，再创建的话，将会自动命名为 App2;
不管是永久节点还是临时节点，均可以在创建时指定参数`-s`使得多次创建相同名称的节点时按照顺序自动递增（这类节点又可以称为sequence节点）。
>5. znode 可以被监控，包括这个目录节点中存储的数据的修改，子节点目录的变化等，一旦变化可以通知设置监控的客户端，这个是 Zookeeper 的核心特性，Zookeeper 的很多功能都是基于这个特性实现的。


#### znode状态信息

1. czxid. 节点创建时的zxid；
2. mzxid. 节点最新一次更新发生时的zxid；
3. ctime. 节点创建时的时间戳；
4. mtime. 节点最新一次更新发生时的时间戳；
5. dataVersion. 节点数据的更新次数；
6. cversion. 其子节点的更新次数；
7. aclVersion. 节点ACL(授权信息)的更新次数；
8. ephemeralOwner. 如果该节点为ephemeral节点, ephemeralOwner值表示与该节点绑定的session id. 如果该节点不是ephemeral节点, ephemeralOwner值为0；
9. dataLength. 节点数据的字节数；
10. numChildren. 子节点个数；

zxid:
ZooKeeper状态的每一次改变, 都对应着一个递增的Transaction(事务) id, 该id称为zxid. 由于zxid的递增性质, 如果zxid1小于zxid2, 那么zxid1肯定先于zxid2发生. 创建任意节点, 或者更新任意节点的数据, 或者删除任意节点, 都会导致Zookeeper状态发生改变, 从而导致zxid的值增加。

### 外在表现：分布式一致性
首先，通过一个简单的演示来加深印象，在本机启动了zk伪集群，启动之后查看状态时可以看到zoo1和zoo3是follwer的角色，zoo2是leader。zoo1对应的客户端端口是2182，zoo2对应的端口是2183，zoo3对应的端口是2184。

![zk伪集群](https://upload-images.jianshu.io/upload_images/3727888-6938c3b376da6521.jpg)

然后通过` zkCli -server 127.0.0.1:2182` 命令连上zoo1，并通过` ls / `命令查看其下的节点，发现只有一个zookeeper节点，然后此时通过`create /test "测试"`命令在根目录下创建一个简单的test节点，可以看到根目录下出现了test节点。
接下来退出zoo1并连接到集群中另外一台zoo2，可以看到zoo2下面也有test节点，同样的zoo3也有，而且内容一致，同样，我在任意一台删除test节点之后，整个集群中所有服务都会删除test节点。

![zk数据一致性](https://upload-images.jianshu.io/upload_images/3727888-4f56bd7bd61f8a8c.jpg)

### 内部机制：选主与广播
#### zookeeper集群中三种角色
1. 领导者（leader），负责进行投票的发起和决议，更新系统状态
2. 学习者（learner），包括跟随者（follower）和观察者（observer），follower用于接受客户端请求并想客户端返回结果，在选主过程中参与投票
3. Observer可以接受客户端连接，将写请求转发给leader，但observer不参加投票过程，只同步leader的状态，observer的目的是为了扩展系统，提高读取速度

#### zookeeper请求类型
对于exists，getData，getChildren等只读请求，收到该请求的zk服务器将会在本地处理，因为每个服务器看到的数据结构内容都是一致的，无所谓在哪台机器上读取数据，因此如果ZooKeeper集群的负载是读多写少，并且读请求分布得均衡的话，效率是很高的。
对于create，setData，delete等有写操作的请求，则需要统一转发给leader处理，leader需要决定编号、执行操作，这个过程称为一个事务（transaction）。

#### 选主过程
当zookeeper集群启动或者leader崩溃额时候，zk进入恢复模式，开始进行选举。
Zk的选举算法有两种：一种是基于basic paxos实现的，另外一种是基于fast paxos算法实现的。系统默认的选举算法为fast paxos。
##### basic paxos流程：
1. 选举线程由当前Server发起选举的线程担任，其主要功能是对投票结果进行统计，并选出推荐的Server；
2. 选举线程首先向所有Server发起一次询问(包括自己)；
3. 选举线程收到回复后，验证是否是自己发起的询问(验证zxid是否一致)，然后获取对方的id(myid)，并存储到当前询问对象列表中，最后获取对方提议的leader相关信息(id,zxid)，并将这些信息存储到当次选举的投票记录表中；
4.  收到所有Server回复以后，就计算出zxid最大的那个Server，并将这个Server相关信息设置成下一次要投票的Server；
5.  线程将当前zxid最大的Server设置为当前Server要推荐的Leader，如果此时获胜的Server获得n/2 + 1的Server票数， 设置当前推荐的leader为获胜的Server，将根据获胜的Server相关信息设置自己的状态，否则，继续这个过程，直到leader被选举出来。

通过流程分析我们可以得出：要使Leader获得多数Server的支持，则Server总数必须是奇数2n+1，且存活的Server的数目不得少于n+1.

##### fast paxos流程
fast paxos流程是在选举过程中，某Server首先向所有Server提议自己要成为leader，当其它Server收到提议以后，解决epoch和 zxid的冲突，并接受对方的提议，然后向对方发送接受提议完成的消息，重复这个流程，最后一定能选举出Leader。

#### 接收工作流程
##### Leader主要有三个功能：
1. 恢复数据；
2. 维持与Learner的心跳，接收Learner请求并判断Learner的请求消息类型；
3. Learner的消息类型主要有PING消息、REQUEST消息、ACK消息、REVALIDATE消息，根据不同的消息类型，进行不同的处理。
PING消息是指Learner的心跳信息；REQUEST消息是Follower发送的提议信息，包括写请求及同步请求；ACK消息是 Follower的对提议的回复，超过半数的Follower通过，则commit该提议；REVALIDATE消息是用来延长SESSION有效时间。

##### Follower主要有四个功能：
1. 向Leader发送请求（PING消息、REQUEST消息、ACK消息、REVALIDATE消息）；
2. 接收Leader消息并进行处理；
3. 接收Client的请求，如果为写请求，发送给Leader进行投票；
4. 返回Client结果。

Follower的消息循环处理如下几种来自Leader的消息：
1. PING消息：心跳消息；
2. PROPOSAL消息：Leader发起的提案，要求Follower投票；
3. COMMIT消息：服务器端最新一次提案的信息；
4. UPTODATE消息：表明同步完成；
5. REVALIDATE消息：根据Leader的REVALIDATE结果，关闭待revalidate的session还是允许其接受消息；
6. SYNC消息：返回SYNC结果到客户端，这个消息最初由客户端发起，用来强制得到最新的更新。

本文参考文章：
1. [ZooKeeper典型应用场景实践](https://my.oschina.net/xianggao/blog/531613)。
2. [zk常用命令](https://www.cnblogs.com/wade-luffy/p/6103956.html)。
3. [zookeeper三种角色](https://blog.csdn.net/mayp1/article/details/52026797)。
4. [zookeeper原理与应用](https://blog.csdn.net/gs80140/article/details/51496925)。