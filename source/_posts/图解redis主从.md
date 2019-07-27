
title: 图解redis主从
date: 2017-10-22 21:15:46
categories: redis进阶
tags: [redis,nosql,源码]
---

### 一、建立连接

当slave收到客户端发来的slave of 命令之后，首先会在redisServer结构体中保存master的host和ip地址，然后建立一个到master的socket连接。
连接成功后，slave会发送PING命令来测试连接是否可用，成功获得响应后，执行REPLCONF命令把自己的listen_port发送到master,后者会在redisClient中包存slave的监听端口。
<!--more-->

![连接][1]
### 二、初次同步数据
连接成功后，slave会发送**PSYNC**命令请求同步数据，由于第一次同步，所以会执行一次**完整重同步**的过程，master收到请求后执行**BGSAVE**命令创建RDB文件[^code]，然后把数据发送到slave 
![full同步][2]
### 三、命令传播（propagate）
同步完成后，master收到的写命令，会由master发送到slave，保证数据同步。命令在master执行成功后会立刻向客户端发送结果，因此命令传播是**异步执行**的，因此redis只能保证主从数据的**最终一致性**。
在命令传播的过程中，master不只把数据发送给slave,还会保存在AOF文件，和复制积压缓冲区(backlog buffer)内。[a]
![此处输入图片的描述][3]
### 四、增量重同步
刚才提到的backlog buffer是一个FIFO的定长队列(环形队列实现)，默认1MB大小，可以通过`repl_backlog_size`配置。它的作用是保存最近的更新命令并记录offsetr。当slave断线重连时，如果能根据offset在buffer中找到丢失的数据,那么只需增量的复制丢失数据即可，不需要进行完成的重同步。
![此处输入图片的描述][4]

### 五、心跳消息
slave定时向master发送REPLCONF命令，通知master当前的自己offset，当master发现slave复制落后时，会主动发送丢失了命令。
心跳消息让master了解slave的连接状态，如果当前slave数量小于min-slave，master会拒绝写命令。info命令输出的lag值反应了slave的上次通信时间。（master也会主动向slave发送PING命令）
```
# Replication
role:master
connected_slaves:2
slave0:ip=127.0.0.1,port=6381,state=online,offset=4518,lag=1
slave1:ip=127.0.0.1,port=6380,state=online,offset=4518,lag=1
```
![heart][5]

### 六、 补充
这是一次重新连接的日志，从服务器发送同步请求，master回复rdb文件。日志中**Partial resynchronization not possible (no cached master)**表示尝试增量同步失败，在我使用redis 2.8.19版本时似乎无法模拟出增量同步的情况，每次断线重连都是完整同步，按照作者说明关闭了slave的aof也不行(同步时不能进行write aof)，[github上这个问题对应的issue][6]
**从服务器**
```[15846] 11 Jan 20:25:34.722 * DB loaded from disk: 2.233 seconds
[15846] 11 Jan 20:25:34.722 * The server is now ready to accept connections on port 6380
[15846] 11 Jan 20:25:35.490 * Connecting to MASTER 127.0.0.1:6379
[15846] 11 Jan 20:25:35.491 * MASTER <-> SLAVE sync started
[15846] 11 Jan 20:25:35.491 * Non blocking connect for SYNC fired the event.
[15846] 11 Jan 20:25:35.491 * Master replied to PING, replication can continue...
[15846] 11 Jan 20:25:35.491 * Partial resynchronization not possible (no cached master)
[15846] 11 Jan 20:25:35.495 * Full resync from master: 13cb06d81ed35702e1cf20dba3aa1159b9a068bd:15931
[15846] 11 Jan 20:25:41.428 * MASTER <-> SLAVE sync: receiving 225990523 bytes from master
[15846] 11 Jan 20:25:44.383 * MASTER <-> SLAVE sync: Flushing old data
[15846] 11 Jan 20:25:44.560 * MASTER <-> SLAVE sync: Loading DB in memory
[15846] 11 Jan 20:25:46.948 * MASTER <-> SLAVE sync: Finished with success```
**主服务器：**
```[14562] 11 Jan 20:23:17.478 # Connection with slave 127.0.0.1:6380 lost.
[14562] 11 Jan 20:23:29.644 # Connection with slave 127.0.0.1:6381 lost.
[14562] 11 Jan 20:25:35.492 * Slave 127.0.0.1:6380 asks for synchronization
[14562] 11 Jan 20:25:35.492 * Full resync requested by slave 127.0.0.1:6380
[14562] 11 Jan 20:25:35.492 * Starting BGSAVE for SYNC with target: disk
[14562] 11 Jan 20:25:35.494 * Background saving started by pid 15849
[14562] 11 Jan 20:25:40.504 * Slave 127.0.0.1:6381 asks for synchronization
[14562] 11 Jan 20:25:40.504 * Full resync requested by slave 127.0.0.1:6381
[14562] 11 Jan 20:25:40.504 * Waiting for end of BGSAVE for SYNC
[15849] 11 Jan 20:25:41.341 * DB saved on disk
[15849] 11 Jan 20:25:41.342 * RDB: 76 MB of memory used by copy-on-write
[14562] 11 Jan 20:25:41.427 * Background saving terminated with success
[14562] 11 Jan 20:25:44.180 * Synchronization with slave 127.0.0.1:6381 succeeded
[14562] 11 Jan 20:25:44.461 * Synchronization with slave 127.0.0.1:6380 succeeded```


[^code]: 当配置设置同步方式为Disk-backed时才会把RDB文件写入磁盘，Diskless模式会直接把RDB文件写到slave的socket


  [1]: /images/redis/conn1.png
  [2]: /images/redis/FULLSYNC.png
  [3]: /images/redis/prpogate2.png
  [4]: /images/redis/incssync.png
  [5]: /images/redis/ping.png
  [6]: https://github.com/antirez/redis/issues/4102