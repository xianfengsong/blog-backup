title: 图解redis sentinel
date: 2017-11-21 23:15:46
categories: redis进阶
tags: [redis,nosql,源码]
---

redis sentinel是redis为支持自动failover的主从模式开发的功能，sentinel实际上也是一个redis实例，不过它只支持部分特有命令，通过`redis-server /path/to/sentinel.conf --sentinel`启动sentinel实例。
<!--more-->

### 整体视图
sentinel服务由多个节点组成，当进行容灾操作时需要多数节点达成一致，因此sentinel节点之间需要建立通信（蓝色部分）。同时为了监控redis节点状态，每个sentinel节点都会和redis实例建立命令连接（图中虚线），sentinel节点还会通过publish命令向其他节点（包括master/slave/sentinel）发送"hello msg"(参考[^sentinelSendPeriodicCommands]）
，通过redis的发布/订阅功能来交换信息。

注意：sentinel接收publish命令时执行的是fake publish
![overview][1]

继续下面的内容前，需要了解sentinel的几个重要配置参数：

>**monitor** master_name,master_ip,master_port,quorum
 (master信息客观下线需要的票数)
**down-after-milliseconds** master_name XXms (失联多久被视为主观下线)
**failover-timeout** master_name XXms (处理故障的超时时间)
**parallel-syncs**  master_name number (执行故障转移时，可以同时从新的master同步数据的slave数量）

### 发现新redis实例
可以发现配置sentinel时不需要填写slave节点的信息，这是因为sentinel可以从它监视的master发送INFO命令获得对应的slave信息，然后再与slave建立连接。通过这种方式sentinel也能够发现新加入的slave节点，过程如图所示：
![1][2]
### 发现新sentinel实例
类似地，一个sentinel实例也能够自主的发现其他sentinel实例，不过这是通过channel接收其他sentinel实例发送的消息实现的，每个sentinel实例会定时所有节点发送PUBLISH命令来向channel投递消息，内容主要是sentinel自己和信息和它保存的master的信息，格式如下：
```
PUBLISH __sentinel__:hello sentinel_ip,sentinel_port,sentinel_runid,current_epoch,
master_name,master_ip,master_port,master_config_epoch.
```
消息会由redis实例的频道发送给所有监视实例的sentinel，其他sentinel收到信息后，如果自己没有保存消息中的sentinel会保存下来，同时epoch如果小于消息中epoch也会自我更新：
![step2][3]

### 判断节点下线过程
sentinel会通过定时向监视的redis实例发送ping命令，检查redis的状态，当redis回复异常或者在超时时间内没有回复时sentinel会将节点设置为主观下线（subjectively down）。而且如果节点是master的话，sentinel需要判断节点是否能设置为客观下线（Objectively down）,过程如图所示：
（1）节点回复不正常，sentinel把节点置为主观下线
（2）给其他sentinel发送`is-master-down-by-addr`命令，格式如下`IS-MASTER-DOWN-BY-ADDR <master_ip> <master_port> <sentinel:current-epoch> <sentinel:runid>`
（只有当发起投票时才会发送runid,此时runid内容为*）
（3）收到`is-master-down-by-addr`的sentinel检查对应redis实例的状态（实际不会发送ping命令，只是检查保存的节点状态是不是主观下线）
（4）发现节点下线回复消息，回复内容如下，后两个是投票时用到的`down state, leader, vote epoch`
（5）当sentinel1 收到足够多的“节点下线”的回复后（大于配置的quorum），将master状态设置为客观下线，然后failover流程才开始进行。

![s3][4]

### failover第一步:选举leader
failover操作只能由一个sentinel节点来执行，因此开始failover首先要选举出一个Leader,sentinel的选举采用的是[Raft算法][5]中的领导人选举机制。在这里确认master客观下线的sentinel会成为候选人，等待failover_start_time（一个随机时间，为了避免候选人同时开始选举导致平票），然后开始向其他实例发起投票请求，依然是使用`is-master-down-by-addr`命令，投票时主要依据是epoch字段。具体流程如图所示：
![s4][6]

### failover第二步：执行故障转移
Leader选举成功之后，会在slave中选择一个最优的，然后“提拔”这个slave作为新的master,然后其他redis实例会来同步新master的数据，具体过程如下：
![dofailover][7]
### sentinel的整体执行流程
上面描述的行为都是通过redis定时任务来驱动的，特别的，sentinel中定义了一个状态机（switch-case代码块）来根据此时master的failover_state执行不同的操作，下面的图从外向内展示了sentinel主要功能的调用过程：
>（1）servCron函数(在server.c)负责执行redis的定时任务，包括过期键，写AOF文件等等，同时包括执行sentinel定时任务
（2）sentinelTimer(sentinel.c)是sentinel模式的定时任务，执行[TILT模式][8]的检查，执行脚本，还有就是处理RedisInstance（集群中所有的master/slave/sentinel实例）
（3）在sentinelHandleRedisInstance函数中，会依次执行reconnect函数维护与其他实例的连接（命令连接和pub/sub）,定期发送命令（info/ping/hello）,判断主观下线
（4）当实例是master时会执行客观下线的检查，在StartFailoverIfNeeded函数中，如果满足条件会开始failover的过程（master会变成SENTINEL_FAILOVER_STATE_WAIT_START，然后还会向其他sentinel确认master状态，使用force模式才会发送is-master-down命令，此时可能也会发起投票）
（5）状态机逻辑的执行在StartFailoverIfNeeded执行之后（和上一步结果无关），状态机就是在每次定时任务循环中检查当前master的状态然后执行不同阶段的failover操作，最后会执行askOther的步骤，本次定时任务结束。


![state][9]
### 附录

**1、完整的failoverstate**
```c 
/* Failover machine different states. */
#define SENTINEL_FAILOVER_STATE_NONE 0  /* No failover in progress. */
#define SENTINEL_FAILOVER_STATE_WAIT_START 1  /* Wait for failover_start_time*/
#define SENTINEL_FAILOVER_STATE_SELECT_SLAVE 2 /* Select slave to promote */
#define SENTINEL_FAILOVER_STATE_SEND_SLAVEOF_NOONE 3 /* Slave -> Master */
#define SENTINEL_FAILOVER_STATE_WAIT_PROMOTION 4 /* Wait slave to change role */
#define SENTINEL_FAILOVER_STATE_RECONF_SLAVES 5 /* SLAVEOF newmaster */
#define SENTINEL_FAILOVER_STATE_UPDATE_CONFIG 6 /* Monitor promoted slave.master配置更新为新的master */
```
**2、redis实例收到的sentinel消息**
集群中一台slave的hello频道中传播的消息，此时master端口是6380：
```
127.0.0.1:6379> psubscribe *
1) "pmessage"
2) "*"
3) "__sentinel__:hello"
4) "127.0.0.1,26380,299da3eb5862baf267d16e36306defe7517bab5b,3,mymaster,127.0.0.1,6380,3"
1) "pmessage"
2) "*"
3) "__sentinel__:hello"
4) "127.0.0.1,26381,bf3ac725160092d5db0fefdcc4814379003a75a8,3,mymaster,127.0.0.1,6380,3"
1) "pmessage"
2) "*"
3) "__sentinel__:hello"
4) "127.0.0.1,26379,9e47039b5fd9735296df2340e54a4c37caeabbb2,3,mymaster,127.0.0.1,6380,3"
```
**3、一次failover的日志**
master port:6380
Leader port:26380
Leader run_id:299da3eb5862baf267d16e36306defe7517bab5b
**follower sentinel收到的消息:**
```
127.0.0.1:26379> psubscribe *
Reading messages... (press Ctrl-C to quit)
1) "psubscribe"
2) "*"
3) (integer) 1
1) "pmessage"
2) "*"
3) "+new-epoch"
4) "2"
1) "pmessage"
2) "*"
3) "+vote-for-leader"
4) "299da3eb5862baf267d16e36306defe7517bab5b 2"
1) "pmessage"
2) "*"
3) "+sdown"
4) "master mymaster 127.0.0.1 6380"
1) "pmessage"
2) "*"
3) "+odown"
4) "master mymaster 127.0.0.1 6380 #quorum 3/2"
1) "pmessage"
2) "*"
3) "+config-update-from"
4) "sentinel 127.0.0.1:26380 127.0.0.1 26380 @ mymaster 127.0.0.1 6380"
1) "pmessage"
2) "*"
3) "+switch-master"
4) "mymaster 127.0.0.1 6380 127.0.0.1 6381"
1) "pmessage"
2) "*"
3) "+slave"
4) "slave 127.0.0.1:6379 127.0.0.1 6379 @ mymaster 127.0.0.1 6381"
1) "pmessage"
2) "*"
3) "+slave"
4) "slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster 127.0.0.1 6381"
1) "pmessage"
2) "*"
3) "+sdown"
4) "slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster 127.0.0.1 6381"
1) "pmessage"
2) "*"
3) "-role-change"
4) "slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster 127.0.0.1 6381 new reported role is master"
1) "pmessage"
2) "*"
3) "-sdown"
4) "slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster 127.0.0.1 6381"
1) "pmessage"
2) "*"
3) "+role-change"
4) "slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster 127.0.0.1 6381 new reported role is slave"

```
sentinel消息各种 + - 标识的含义:
>+reset-master <instance details> -- The master was reset.
    +slave <instance details> -- A new slave was detected and attached.
    +failover-state-reconf-slaves <instance details> -- Failover state changed to reconf-slaves state.
    +failover-detected <instance details> -- A failover started by another Sentinel or any other external entity was detected (An attached slave turned into a master).
    +slave-reconf-sent <instance details> -- The leader sentinel sent the SLAVEOF command to this instance in order to reconfigure it for the new slave.
    +slave-reconf-inprog <instance details> -- The slave being reconfigured showed to be a slave of the new master ip:port pair, but the synchronization process is not yet complete.
    +slave-reconf-done <instance details> -- The slave is now synchronized with the new master.
    -dup-sentinel <instance details> -- One or more sentinels for the specified master were removed as duplicated (this happens for instance when a Sentinel instance is restarted).
    +sentinel <instance details> -- A new sentinel for this master was detected and attached.
    +sdown <instance details> -- The specified instance is now in Subjectively Down state.
    -sdown <instance details> -- The specified instance is no longer in Subjectively Down state.
    +odown <instance details> -- The specified instance is now in Objectively Down state.
    -odown <instance details> -- The specified instance is no longer in Objectively Down state.
    +new-epoch <instance details> -- The current epoch was updated.
    +try-failover <instance details> -- New failover in progress, waiting to be elected by the majority.
    +elected-leader <instance details> -- Won the election for the specified epoch, can do the failover.
    +failover-state-select-slave <instance details> -- New failover state is select-slave: we are trying to find a suitable slave for promotion.
    no-good-slave <instance details> -- There is no good slave to promote. Currently we'll try after some time, but probably this will change and the state machine will abort the failover at all in this case.
    selected-slave <instance details> -- We found the specified good slave to promote.
    failover-state-send-slaveof-noone <instance details> -- We are trying to reconfigure the promoted slave as master, waiting for it to switch.
    failover-end-for-timeout <instance details> -- The failover terminated for timeout, slaves will eventually be configured to replicate with the new master anyway.
    failover-end <instance details> -- The failover terminated with success. All the slaves appears to be reconfigured to replicate with the new master.
    switch-master <master name> <oldip> <oldport> <newip> <newport> -- The master new IP and address is the specified one after a configuration change. This is the message most external users are interested in.
    +tilt -- Tilt mode entered.
    -tilt -- Tilt mode exited.

**leader 日志**
```
[30369] 13 Jan 20:21:06.701 # +sdown master mymaster 127.0.0.1 6380
[30369] 13 Jan 20:21:06.768 # +odown master mymaster 127.0.0.1 6380 #quorum 2/2
[30369] 13 Jan 20:21:06.768 # +new-epoch 2
[30369] 13 Jan 20:21:06.768 # +try-failover master mymaster 127.0.0.1 6380
[30369] 13 Jan 20:21:06.782 # +vote-for-leader 299da3eb5862baf267d16e36306defe7517bab5b 2
[30369] 13 Jan 20:21:06.821 # 127.0.0.1:26381 voted for 299da3eb5862baf267d16e36306defe7517bab5b 2
[30369] 13 Jan 20:21:06.821 # 127.0.0.1:26379 voted for 299da3eb5862baf267d16e36306defe7517bab5b 2
[30369] 13 Jan 20:21:06.844 # +elected-leader master mymaster 127.0.0.1 6380
[30369] 13 Jan 20:21:06.844 # +failover-state-select-slave master mymaster 127.0.0.1 6380
[30369] 13 Jan 20:21:06.902 # +selected-slave slave 127.0.0.1:6381 127.0.0.1 6381 @ mymaster 127.0.0.1 6380
[30369] 13 Jan 20:21:06.902 * +failover-state-send-slaveof-noone slave 127.0.0.1:6381 127.0.0.1 6381 @ mymaster 127.0.0.1 6380
[30369] 13 Jan 20:21:06.965 * +failover-state-wait-promotion slave 127.0.0.1:6381 127.0.0.1 6381 @ mymaster 127.0.0.1 6380
[30369] 13 Jan 20:21:07.830 # +promoted-slave slave 127.0.0.1:6381 127.0.0.1 6381 @ mymaster 127.0.0.1 6380
[30369] 13 Jan 20:21:07.830 # +failover-state-reconf-slaves master mymaster 127.0.0.1 6380
[30369] 13 Jan 20:21:07.872 * +slave-reconf-sent slave 127.0.0.1:6379 127.0.0.1 6379 @ mymaster 127.0.0.1 6380
[30369] 13 Jan 20:21:08.852 * +slave-reconf-inprog slave 127.0.0.1:6379 127.0.0.1 6379 @ mymaster 127.0.0.1 6380
[30369] 13 Jan 20:21:08.976 # -odown master mymaster 127.0.0.1 6380
[30369] 13 Jan 20:22:53.198 * +reboot master mymaster 127.0.0.1 6380
[30369] 13 Jan 20:22:53.249 # -sdown master mymaster 127.0.0.1 6380

```
**follower 日志**
```
[29838] 13 Jan 20:21:06.803 # +new-epoch 2
[29838] 13 Jan 20:21:06.821 # +vote-for-leader 299da3eb5862baf267d16e36306defe7517bab5b 2
[29838] 13 Jan 20:21:06.821 # +sdown master mymaster 127.0.0.1 6380
[29838] 13 Jan 20:21:06.898 # +odown master mymaster 127.0.0.1 6380 #quorum 3/2
[29838] 13 Jan 20:21:06.898 # Next failover delay: I will not start a failover before Sat Jan 13 20:27:07 2018
[29838] 13 Jan 20:21:07.874 # +config-update-from sentinel 127.0.0.1:26380 127.0.0.1 26380 @ mymaster 127.0.0.1 6380
[29838] 13 Jan 20:21:07.874 # +switch-master mymaster 127.0.0.1 6380 127.0.0.1 6381
[29838] 13 Jan 20:21:07.874 * +slave slave 127.0.0.1:6379 127.0.0.1 6379 @ mymaster 127.0.0.1 6381
[29838] 13 Jan 20:21:07.894 * +slave slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster 127.0.0.1 6381
[29838] 13 Jan 20:21:37.913 # +sdown slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster 127.0.0.1 6381
[29838] 13 Jan 20:22:53.322 # -sdown slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster 127.0.0.1 6381
```
[^sentinelSendPeriodicCommands]:https://github.com/antirez/redis/blob/3.0/src/sentinel.c sentinelSendPeriodicCommands()


  [1]: http://7xl4v5.com1.z0.glb.clouddn.com/sentinel.png
  [2]: http://7xl4v5.com1.z0.glb.clouddn.com/redis/sentinel/s1.png
  [3]: http://7xl4v5.com1.z0.glb.clouddn.com/redis/sentinel/s2.png
  [4]: http://7xl4v5.com1.z0.glb.clouddn.com/redis/sentinel/s3.png
  [5]: http://www.infoq.com/cn/articles/raft-paper
  [6]: http://7xl4v5.com1.z0.glb.clouddn.com/redis/sentinel/s4.png
  [7]: http://7xl4v5.com1.z0.glb.clouddn.com/redis/sentinel/s5.png
  [8]: http://doc.redisfans.com/topic/sentinel.html#tilt
  [9]: http://7xl4v5.com1.z0.glb.clouddn.com/redis/sentinel/state.png