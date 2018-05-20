---
title: zookeeper安装与伪集群配置
date: 2018-05-20 18:53:24
---
在自己学习分布式相关的技术时往往需要使用zookeeper伪集群配置，故记录于此。

### 安装ZK
本文操作系统是Mac OS，为了便于各种软件的管理，使用homebrew安装。
#### 安装
	使用homebrew进行安装zookeeper,命令：brew install zookeeper
	安装完成之后zookeeper配置文件位置：/usr/local/etc/zookeeper
	安装目录位置：/usr/local/Cellar/zookeeper

#### 常用命令（读取默认的配置文件zoo.cfg）
	启动服务：zkServer start
	停止服务：zkServer stop
	查看服务状态：zkServer status
	连接: zkCli


#### 伪集群配置
根据/usr/local/etc/zookeeper下的zoo.cfg中的内容，创建zoo1.cfg、zoo2.cfg、zoo3.cfg，（懒人笔记，复制创建：cp zoo.cfg zoo1.cfg）
如果没有zoo.cfg，可以从zoo_sample.cfg取得内容

配置内容如下：
```text
# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial 
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes.
# 自定义的zk数据目录，zk+对应的服务序号
dataDir=/usr/local/var/run/zookeeper/data/zk1
dataLogDir=/usr/local/var/run/zookeeper/data/zk1/logs
# the port at which the clients will connect
clientPort=2182
# 伪集群配置
server.1=127.0.0.1:2888:3888
server.2=127.0.0.1:2888:3889
server.3=127.0.0.1:2888:3890
# the maximum number of client connections.
# increase this if you need to handle more clients
#maxClientCnxns=60
#
# Be sure to read the maintenance section of the 
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1
```
	
配置完成后，前往dataDir（配置文件zoo+序号.cfg中的dataDir）创建zk1、zk2、zk3目录（懒人笔记 批量创建：mkdir zk1 zk2 zk3）,分别在对应目录下创建myid文件，并输入对应的序号，例如在zk1目录下vim myid，然后输入1退出即可

#### 启动集群
命令同上文第2步，在后面加上对应的配置文件即可（我这里是在zoo1.cfg所在目录中执行的）
```text
zkServer start zoo1.cfg
zkServer start zoo2.cfg
zkServer start zoo3.cfg
```
这里有一个点就是，因为我配置的是三个zk服务，所以只要当三台中正在运行的服务数少于(3+1)/2时，用zkServer status zoox.cfg命令查看相应的zk服务的状态都会显示`“Error contacting service. It is probably not running.”`，但是此时不代表zoox.cfg对应的服务没有启动，而是zk整个集群不可用。

比如我首先启动zoo1.cfg对应的服务，然后我输入`zkServer status zoo1.cfg`查看状态，会显示:
```text 
ZooKeeper JMX enabled by default
Using config: /usr/local/etc/zookeeper/zoo1.cfg
Error contacting service. It is probably not running.
```
但是使用zkCli命令`zkCli -server 127.0.0.1:2182`其实可以连接到zoo1.cfg对应的zk服务的。
接下来启动zoo2.cfg之后，再查看zoo1.cfg和zoo2.cfg的状态：
```text
192:zookeeper shakeli$ zkServer status zoo1.cfg
ZooKeeper JMX enabled by default
Using config: /usr/local/etc/zookeeper/zoo1.cfg
Mode: follower
192:zookeeper shakeli$ zkServer status zoo2.cfg 
ZooKeeper JMX enabled by default
Using config: /usr/local/etc/zookeeper/zoo2.cfg
Mode: leader
```
可以看出两个ZK服务的状态是正常的，zoo1.cfg是follower，zoo2.cfg是leader
这个跟正常的集群模式中在单个启动时会报错是同样的原因。








