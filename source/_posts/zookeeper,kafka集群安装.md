# zookeeper,kafka集群安装

title: zookeeper,kafka集群安装
date: 2017-07-26 00:09:46
categories: 
tags: [kafaka,zookeeper,集群]

---

系统版本 CentOS 6.6
java version 1.7
zookeeper版本 3.4.10
kafka版本 0.11.0.1

## 安装zookeeper

### 1.下载
```bash
wget https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/zookeeper-3.4.10/zookeeper-3.4.10.tar.gz
tar -xvf zookeeper-3.4.10.tar.gz
```
<!--more-->

### 2.修改配置文件


把zoo_sample.cfg改成zoo.cfg
```bash
cd zookeeper-3.4.10/conf
mv zoo_sample.cfg zoo.cfg
```
配置文件说明：

>tickTime=2000
dataDir=/var/lib/zookeeper
clientPort=2181
initLimit=5
syncLimit=2
server.1=192.168.185.153:2888:3888
server.2=192.168.185.154:2888:3888
server.3=10.252.81.25:2888:3888


- **tickTime**: 默认2000ms,zk的基本时间单位，用来做发送心跳消息的时间间隔，会话超时的最小时间是2×tickTime.
- **dataDir**: 存储内存数据库快照的位置，也是更新数据库的事务日志的默认存储位置
- **clientPort**: 给客户端连接的端口
- **initLimit**: 用来配置 Zookeeper Leader接受Follower初始化连接时最长能忍受多少个心跳时间间隔数,默认5，即5×tickTime=10000ms
- **syncLimit**:  Leader 与 Follower 之间发送消息，请求和应答时间长度，最长不能超过多少个 tickTime 的时间长度，总的时间长度就是 2*2000=4 秒
- **server.x**: server.id=ip:port1:port2,使用伪集群模式时可以填写相同的ip（但是端口要不同）,port1是集群中实例通信的端口，port2是用来选举leader的端口

在每个机器依次执行 1 2 两步，并使用相同的配置文件。

### 3.创建myid文件
在配置的dataDir目录创建一个名为myid的文件，文件内容是配置的server.id的id对应的数字，用来告知机器它代表的server。
```
echo "3">/var/lib/zookeeper/myid
```
最后执行 `bash bin/zkServer.sh start` 逐个启动zk

### 4.测试
执行`bin/zkCli.sh -server 127.0.0.1:2181`启动客户端建立连接
看到`Welcome to ZooKeeper!`表示连接成功。

## 安装kafka
在安装并启动zookeeper之后，就可以安装kafka了。
### 1.下载
```bash
wget http://mirror.bit.edu.cn/apache/kafka/0.11.0.1/kafka_2.11-0.11.0.1.tgz
tar -xzf kafka_2.11-0.11.0.1.tgz
```
### 2.修改配置文件
conf/server.properties文件 
默认配置简要介绍如下，[完整配置参考](https://kafka.apache.org/documentation/#configuration)：
```bash
# broker的唯一id
broker.id=0

# 接收请求发送响应的线程数
numnetwork.threads=3

# 处理请求的线程数（包含磁盘i/o）
num.io.threads=8
# The send buffer (SO_SNDBUF) used by the socket server
socket.send.buffer.bytes=102400
# The receive buffer (SO_RCVBUF) used by the socket server
socket.receive.buffer.bytes=102400
# The maximum size of a request that the socket server will accept (protection against OOM)
socket.request.max.bytes=104857600

#Log Basics #
# 保存日志文件位置
log.dirs=/tmp/kafka-logs
# 每个主题默认的分区数
num.partitions=1
# 每个数据目录的数量，用于启动时的日志恢复和关机时的flush
# 对于具有位于RAID阵列中的数据目录的安装，建议增加此值。
num.recovery.threads.per.data.dir=1

#Internal Topic Settings  #############################
# offset topic的备份数   
offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1

##Log Retention Policy 日志保留策略##################

# 至少保留多久
log.retention.hours=168

# 最大日志，超过会划分新日志
log.segment.bytes=1073741824

# 检查日志是否删除的时间间隔
log.retention.check.interval.ms=300000

## Zookeeper #####

# Zookeeper 连接地址
zookeeper.connect=localhost:2181
# Zookeeper 连接 Timeout 
zookeeper.connection.timeout.ms=6000


### Group Coordinator Settings####################
#0.11.0.0引入的新配置，指定当新成员加入组时，GroupCoordinator将延迟初始消费者重新平衡（rebalance）的时间（毫秒），重新平衡将进一步延迟group.initial.rebalance.delay.ms的值，最大值为max.poll.interval.ms。
#测试环境配置为0减少等待时间，生产环境建议配置3秒
group.initial.rebalance.delay.ms=0
```
修改这几个字段
>broker.id=yourid
num.parttions=3
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2
zookeeper.connect=192.168.185.153:2181,192.168.185.154:2181,10.252.81.25:2181

### 3.启动kafka
```bash
bin/kafka-server-start.sh config/server.properties &
```
### 4.测试

创建话题"hello-kafka",复制2份，使用2个partition
```bash
>bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 2 --partitions 2 --topic hello-kafka
```
>Created topic "hello-kafka".

查看kafka状态
```bash
bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic hello-kafka
```
输出
>Topic:hello-kafka	PartitionCount:2	ReplicationFactor:2	Configs:
	Topic: hello-kafka	Partition: 0	Leader: 3	Replicas: 3,2	Isr: 3,2
	Topic: hello-kafka	Partition: 1	Leader: 1	Replicas: 1,3	Isr: 1

连接zookeeper服务端，看到kafka在zookeeper保存的话题等信息
```bash
[zk: 127.0.0.1:2181(CONNECTED) 2] ls /
[isr_change_notification, zookeeper, admin, consumers, cluster, config, latest_producer_id_block, controller, brokers, controller_epoch]

[zk: 127.0.0.1:2181(CONNECTED) 5] ls /brokers/topics
[hello, my-test, my-replicated-topi, my-replicated-topic, hello-kafka]
```


参考
zookeeper介绍 ：https://www.ibm.com/developerworks/cn/opensource/os-cn-zookeeper/
zookeeper官方文档 ：http://zookeeper.apache.org/doc/r3.4.10/zookeeperStarted.html
kafka官方文档 ：https://kafka.apache.org/documentation/
kafka文档翻译 ：http://www.orchome.com/472



