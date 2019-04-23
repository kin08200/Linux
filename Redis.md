#一、Redis主从配置
## 1、环境说明

| 主机名称      | IP地址        | redis版本和角色说明 |
| ------------- | ------------- | ------------------- |
| redis-master  | 192.168.56.11 | redis 5.0.3（主）   |
| redis-slave01 | 192.168.56.12 | redis 5.0.3（从）   |
| redis-slave02 | 192.168.56.13 | redis 5.0.3（从）   |

## 2、修改主从的redis配置文件

```bash
[root@redis-master ~]# grep -Ev "^$|#" /usr/local/redis/redis.conf 
bind 192.168.56.11
protected-mode yes
port 6379
daemonize yes
pidfile /var/run/redis_6379.pid
logfile "/var/log/redis.log"
dir /var/redis/

[root@redis-slave01 ~]# grep -Ev "^$|#" /usr/local/redis/redis.conf
bind 192.168.56.12
protected-mode yes
port 6379
daemonize yes
pidfile /var/run/redis_6379.pid
logfile "/var/log/redis.log"
dir /var/redis/
replicaof 192.168.56.11 6379	#配置为master的从，如果master上有密码配置，还需要增加下面一项密码配置
masterauth 123456	#配置主的密码

[root@redis-slave02 ~]# grep -Ev "^$|#" /usr/local/redis/redis.conf
bind 192.168.56.13
protected-mode yes
port 6379
daemonize yes
pidfile /var/run/redis_6379.pid
logfile "/var/log/redis.log"
dir /var/redis/
replicaof 192.168.56.11 6379	#配置为master的从
masterauth 123456	#配置主的密码
```
##3、启动主从redis
这里需要注意的是：redis主从和mysql主从不一样，redis主从不用事先同步数据，它会自动同步过去
```bash
[root@redis-master ~]# systemctl start redis
[root@redis-slave01 ~]# systemctl start redis
[root@redis-slave02 ~]# systemctl start redis
[root@redis-master ~]# netstat -tulnp |grep redis
tcp        0      0 192.168.56.11:6379      0.0.0.0:*               LISTEN      1295/redis-server 1 
[root@redis-slave01 ~]# netstat -tulnp |grep redis
tcp        0      0 192.168.56.12:6379      0.0.0.0:*               LISTEN      1625/redis-server 1 
[root@redis-slave02 ~]# netstat -tulnp |grep redis
tcp        0      0 192.168.56.13:6379      0.0.0.0:*               LISTEN      1628/redis-server 1 
```
##3、数据同步验证
```bash
[root@redis-master ~]# redis-cli -h 192.168.56.11	#主上写入数据
192.168.56.11:6379> KEYS *
(empty list or set)
192.168.56.11:6379> set k1 123
OK
192.168.56.11:6379> set k2 456
OK

[root@redis-slave01 ~]# redis-cli -h 192.168.56.12	#slave01上查看是否数据同步
192.168.56.12:6379> KEYS *
1) "k2"
2) "k1"
192.168.56.12:6379> get k1
"123"
192.168.56.12:6379> get k2
"456"

[root@redis-slave02 ~]# redis-cli -h 192.168.56.13	#slave02上查看是否数据同步
192.168.56.13:6379> KEYS *
1) "k2"
2) "k1"
192.168.56.13:6379> get k1
"123"
192.168.56.13:6379> get k2
"456"
```

# 二、Redis哨兵模式

## 1、Redis sentinel介绍

Redis Sentinel是Redis高可用的实现方案。Sentinel是一个管理多个Redis实例的工具，它可以实现对Redis的监控、通知、自动故障转移。

##2、Redis Sentinel的主要功能

Sentinel的主要功能包括主节点存活检测、主从运行情况检测、自动故障转移（failover）、主从切换。Redis的Sentinel最小配置是一主一从。 Redis的Sentinel系统可以用来管理多个Redis服务器，该系统可以执行以下四个任务：

- 监控

  Sentinel会不断的检查主服务器和从服务器是否正常运行。

- 通知

  当被监控的某个Redis服务器出现问题，Sentinel通过API脚本向管理员或者其他的应用程序发送通知。

- 自动故障转移

  当主节点不能正常工作时，Sentinel会开始一次自动的故障转移操作，它会将与失效主节点是主从关系的其中一个从节点升级为新的主节点， 并且将其他的从节点指向新的主节点。

- 配置提供者

  在Redis Sentinel模式下，客户端应用在初始化时连接的是Sentinel节点集合，从中获取主节点的信息。

##3、Redis Sentinel的工作流程

Sentinel是Redis的高可用性解决方案：

由一个或多个Sentinel实例组成的Sentinel系统可以监视任意多个主服务器，以及所有从服务器，并在被监视的主服务器进入下线状态时，自动将下线主服务器属下的某个从服务器升级为新的主服务器，然后由新的主服务器代替已下线的主服务器继续处理命令请求 。

## 4、相关概念

- 主观失效

  SDOWN（subjectively down）,直接翻译的为”主观”失效,即当前sentinel实例认为某个redis服务为”不可用”状态.

- 客观失效

  ODOWN（objectively down）,直接翻译为”客观”失效,即多个sentinel实例都认为master处于”SDOWN”状态,那么此时master将处于ODOWN,ODOWN可以简单理解为master已经被集群确定为”不可用”,将会开启failover

## 5、环境说明

| 主机名称      | IP地址              | redis版本和角色说明 |
| ------------- | ------------------- | ------------------- |
| redis-master  | 192.168.56.11:6379  | redis 5.0.3（主）   |
| redis-slave01 | 192.168.56.12:6379  | redis 5.0.3（从）   |
| redis-slave02 | 192.168.56.13:6379  | redis 5.0.3（从）   |
| redis-master  | 192.168.56.11:26379 | Sentinel01          |
| redis-slave01 | 192.168.56.12:26379 | Sentinel02          |
| redis-slave02 | 192.168.56.13:26379 | Sentinel03          |

## 6、部署Sentinel

Sentinel.conf配置文件主要参数解析：

```bash
# 端口
port 26379

# 是否后台启动
daemonize yes

# pid文件路径
pidfile /var/run/redis-sentinel.pid

# 日志文件路径
logfile "/var/log/sentinel.log"

# 定义工作目录
dir /tmp

# 定义Redis主的别名, IP, 端口，这里的2指的是需要至少2个Sentinel认为主Redis挂了才最终会采取下一步行为
sentinel monitor mymaster 127.0.0.1 6379 2

# 如果mymaster 30秒内没有响应，则认为其主观失效
sentinel down-after-milliseconds mymaster 30000

# 如果master重新选出来后，其它slave节点能同时并行从新master同步数据的台数有多少个，显然该值越大，所有slave节点完成同步切换的整体速度越快，但如果此时正好有人在访问这些slave，可能造成读取失败，影响面会更广。最保守的设置为1，同一时间，只能有一台干这件事，这样其它slave还能继续服务，但是所有slave全部完成缓存更新同步的进程将变慢。
sentinel parallel-syncs mymaster 1

# 该参数指定一个时间段，在该时间段内没有实现故障转移成功，则会再一次发起故障转移的操作，单位毫秒
sentinel failover-timeout mymaster 180000

# 不允许使用SENTINEL SET设置notification-script和client-reconfig-script。
sentinel deny-scripts-reconfig yes
```

修改三台Sentinel的配置文件，如下

```bash
[root@redis-master ~]# grep -Ev "^$|#" /usr/local/redis/sentinel.conf 
port 26379
daemonize yes
pidfile "/var/run/redis-sentinel.pid"
logfile "/var/log/sentinel.log"
dir "/tmp"
sentinel monitor mymaster 192.168.56.11 6379 2
sentinel down-after-milliseconds mymaster 30000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 180000
sentinel deny-scripts-reconfig yes

[root@redis-slave01 ~]# grep -Ev "^$|#" /usr/local/redis/sentinel.conf 
port 26379
daemonize yes
pidfile "/var/run/redis-sentinel.pid"
logfile "/var/log/sentinel.log"
dir "/tmp"
sentinel monitor mymaster 192.168.56.11 6379 2
sentinel down-after-milliseconds mymaster 30000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 180000
sentinel deny-scripts-reconfig yes

[root@redis-slave02 ~]# grep -Ev "^$|#" /usr/local/redis/sentinel.conf 
port 26379
daemonize yes
pidfile "/var/run/redis-sentinel.pid"
logfile "/var/log/sentinel.log"
dir "/tmp"
sentinel monitor mymaster 192.168.56.11 6379 2
sentinel down-after-milliseconds mymaster 30000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 180000
sentinel deny-scripts-reconfig yes
```

## 7、启动Sentinel

启动的顺序：主Redis --> 从Redis --> Sentinel1/2/3

```bash
[root@redis-master ~]# redis-sentinel /usr/local/redis/sentinel.conf 
[root@redis-master ~]# ps -ef |grep redis
root      1295     1  0 14:03 ?        00:00:06 /usr/local/redis/src/redis-server 192.168.56.11:6379
root      1407     1  1 14:40 ?        00:00:00 redis-sentinel *:26379 [sentinel]
root      1412  1200  0 14:40 pts/1    00:00:00 grep --color=auto redis

[root@redis-slave01 ~]# redis-sentinel /usr/local/redis/sentinel.conf 
[root@redis-slave01 ~]# ps -ef |grep redis
root      1625     1  0 14:04 ?        00:00:06 /usr/local/redis/src/redis-server 192.168.56.12:6379
root      1715     1  1 14:41 ?        00:00:00 redis-sentinel *:26379 [sentinel]
root      1720  1574  0 14:41 pts/0    00:00:00 grep --color=auto redis

[root@redis-slave02 ~]# redis-sentinel /usr/local/redis/sentinel.conf 
[root@redis-slave02 ~]# ps -ef |grep redis
root      1628     1  0 14:07 ?        00:00:06 /usr/local/redis/src/redis-server 192.168.56.13:6379
root      1709     1  0 14:42 ?        00:00:00 redis-sentinel *:26379 [sentinel]
root      1714  1575  0 14:42 pts/0    00:00:00 grep --color=auto redis
```

## 8、Sentinel操作

```bash
[root@redis-master ~]# redis-cli -p 26379	#哨兵模式查看
127.0.0.1:26379> sentinel master mymaster	#输出被监控的主节点的状态信息
 1) "name"
 2) "mymaster"
 3) "ip"
 4) "192.168.56.11"
 5) "port"
 6) "6379"
 7) "runid"
 8) "bae06cc3bc6dcbff7c2de1510df7faf1a6eb6941"
 9) "flags"
10) "master"
......
127.0.0.1:26379> sentinel slaves mymaster	#查看mymaster的从信息，可以看到有2个从节点
1)  1) "name"
    2) "192.168.56.12:6379"
    3) "ip"
    4) "192.168.56.12"
    5) "port"
    6) "6379"
    7) "runid"
    8) "c86027e7bdd217cb584b1bd7a6fea4ba79cf6364"
    9) "flags"
   10) "slave"
......
2)  1) "name"
    2) "192.168.56.13:6379"
    3) "ip"
    4) "192.168.56.13"
    5) "port"
    6) "6379"
    7) "runid"
    8) "61597fdb615ecf8bd7fc18e143112401ed6156ec"
    9) "flags"
   10) "slave"
......

127.0.0.1:26379> sentinel sentinels mymaster	#查看其它sentinel信息
1)  1) "name"
    2) "ba12e2a4023d2e9bcad282395ba6b14030920070"
    3) "ip"
    4) "192.168.56.12"
    5) "port"
    6) "26379"
    7) "runid"
    8) "ba12e2a4023d2e9bcad282395ba6b14030920070"
    9) "flags"
   10) "sentinel"
......
2)  1) "name"
    2) "14fca3f851e9e1bd3a4a0dc8a9e34bb237648455"
    3) "ip"
    4) "192.168.56.13"
    5) "port"
    6) "26379"
    7) "runid"
    8) "14fca3f851e9e1bd3a4a0dc8a9e34bb237648455"
    9) "flags"
   10) "sentinel"
```

## 9、哨兵模式下的主从测试

模拟停止master上的Redis，查看Redis的主从变化，如下：

```bash
[root@redis-master ~]# systemctl stop redis		#停止master上的redis

[root@redis-slave01 ~]# tail -n 20 /var/log/sentinel.log 	#查看哨兵日志
......
1747:X 19 Apr 2019 14:59:01.747 # +monitor master mymaster 192.168.56.11 6379 quorum 2
1747:X 19 Apr 2019 14:59:44.829 # +sdown sentinel 14fca3f851e9e1bd3a4a0dc8a9e34bb237648455 192.168.56.13 26379 @ mymaster 192.168.56.11 6379
1747:X 19 Apr 2019 14:59:46.950 # -sdown sentinel 14fca3f851e9e1bd3a4a0dc8a9e34bb237648455 192.168.56.13 26379 @ mymaster 192.168.56.11 6379
1747:X 19 Apr 2019 15:00:44.391 # +sdown master mymaster 192.168.56.11 6379
1747:X 19 Apr 2019 15:00:44.525 # +new-epoch 1
1747:X 19 Apr 2019 15:00:44.527 # +vote-for-leader 14fca3f851e9e1bd3a4a0dc8a9e34bb237648455 1
1747:X 19 Apr 2019 15:00:45.023 # +config-update-from sentinel 14fca3f851e9e1bd3a4a0dc8a9e34bb237648455 192.168.56.13 26379 @ mymaster 192.168.56.11 6379
1747:X 19 Apr 2019 15:00:45.023 # +switch-master mymaster 192.168.56.11 6379 192.168.56.13 6379
1747:X 19 Apr 2019 15:00:45.024 * +slave slave 192.168.56.12:6379 192.168.56.12 6379 @ mymaster 192.168.56.13 6379
1747:X 19 Apr 2019 15:00:45.024 * +slave slave 192.168.56.11:6379 192.168.56.11 6379 @ mymaster 192.168.56.13 6379
1747:X 19 Apr 2019 15:01:15.050 # +sdown slave 192.168.56.11:6379 192.168.56.11 6379 @ mymaster 192.168.56.13 6379

#从上面的日志可以看到master已经sdown，并切换为192.168.56.13为master节点，下面查看slave01上的配置，会自动的更改replicaof配置项，如下：

[root@redis-slave01 ~]# grep "replicaof" /usr/local/redis/redis.conf |grep -vE "#"
replicaof 192.168.56.13 6379

[root@redis-master ~]# redis-cli -p 26379	#哨兵模式下查看主从信息，也是可以看到主从的变化
127.0.0.1:26379> sentinel master mymaster
 1) "name"
 2) "mymaster"
 3) "ip"
 4) "192.168.56.13"
 5) "port"
 6) "6379"
 7) "runid"
 8) "61597fdb615ecf8bd7fc18e143112401ed6156ec"
 9) "flags"
10) "master"
......
127.0.0.1:26379> sentinel slaves mymaster
1)  1) "name"
    2) "192.168.56.12:6379"
    3) "ip"
    4) "192.168.56.12"
    5) "port"
    6) "6379"
    7) "runid"
    8) "c86027e7bdd217cb584b1bd7a6fea4ba79cf6364"
    9) "flags"
   10) "slave"
......
2)  1) "name"
    2) "192.168.56.11:6379"
    3) "ip"
    4) "192.168.56.11"
    5) "port"
    6) "6379"
    7) "runid"
    8) ""
    9) "flags"
   10) "s_down,slave,disconnected"	#提示该节点为从，并且状态为s_down，无法链接的状态
   ......
```

# 三、Redis集群

## 1、Redis Cluster简介

Redis Cluster为Redis官方提供的一种分布式集群解决方案。它支持在线节点增加和减少。 集群中的节点角色可能是主，也可能是从，但需要保证每个主节点都要有对应的从节点， 这样保证了其高可用。

Redis Cluster采用了分布式系统的分片（分区）的思路，每个主节点为一个分片，这样也就意味着 存储的数据是分散在所有分片中的。当增加节点或删除主节点时，原存储在某个主节点中的数据 会自动再次分配到其他主节点。

如下图，各节点间是相互通信的，通信端口为各节点Redis服务端口+10000，这个端口是固定的，所以注意防火墙设置， 节点之间通过二进制协议通信，这样的目的是减少带宽消耗。

在Redis Cluster中有一个概念slot，我们翻译为槽。Slot数量是固定的，为16384个。这些slot会均匀地分布到各个 节点上。另外Redis的键和值会根据hash算法存储在对应的slot中。简单讲，对于一个键值对，存的时候在哪里是通过 hash算法算出来的，那么取得时候也会算一下，知道值在哪个slot上。根据slot编号再到对应的节点上去取。

Redis Cluster无法保证数据的强一致性，这是因为当数据存储时，只要在主节点上存好了，就会告诉客户端存好了， 如果等所有从节点上都同步完再跟客户端确认，那么会有很大的延迟，这个对于客户端来讲是无法容忍的。所以， 最终Redis Cluster只好放弃了数据强一致性，而把性能放在了首位。

![img](https://github.com/aminglinux/linux2019/raw/master/images/redis_cluster.png?raw=true)

## 2、Redis Cluster环境说明

Redis Cluster至少需要三个节点，即一主二从，本实验中我们使用6个节点搭建。

| 主机名        | IP+Port            | 角色         |
| ------------- | ------------------ | ------------ |
| redis-master  | 192.168.56.11:6379 | Redis Master |
| redis-slave01 | 192.168.56.12:6379 | Redis Master |
| redis-slave02 | 192.168.56.13:6379 | Redis Master |
| redis-master  | 192.168.56.11:6380 | Redis Slave  |
| redis-slave01 | 192.168.56.12:6380 | Redis Slave  |
| redis-slave02 | 192.168.56.13:6380 | Redis Slave  |

## 3、Redis部署

这里使用三台虚拟机，每台虚拟机运行两个Redis实例，端口号分别为6379和6380，在进行修改配置文件时，需要将之前的哨兵模式和主从配置项取消掉，并且需要将原来的数据存储目录中的数据清空。否则Redis是无法启动集群模式的。下面给出其中一主一从的6379和6380的部分需要修改的配置，其余两个节点采用一样的配置即可。

```bash
[root@redis-master ~]# grep -Ev "^$|#" /usr/local/redis/redis.conf 
bind 192.168.56.11
protected-mode yes
port 6379
tcp-backlog 511
timeout 0
tcp-keepalive 300
daemonize yes
pidfile "/var/run/redis_6379.pid"
logfile "/var/log/redis.log"
dir "/var/redis"
cluster-enabled yes		#开启集群
cluster-config-file nodes-6379.conf		#集群的配置文件，首次启动会自动创建
cluster-node-timeout 15000		#集群节点连接超时时间，15秒
......

[root@redis-master ~]# grep -Ev "^$|#" /usr/local/redis/redis_6380.conf 
bind 192.168.56.11
protected-mode yes
port 6380
tcp-backlog 511
timeout 0
tcp-keepalive 300
daemonize yes
pidfile "/var/run/redis_6380.pid"
logfile "/var/log/redis_6380.log"
dir "/var/redis_6380"
cluster-enabled yes		#开启集群
cluster-config-file nodes-6380.conf		#集群的配置文件，首次启动会自动创建
cluster-node-timeout 15000		#集群节点连接超时时间，15秒
......

[root@redis-master ~]# mkdir /var/redis_6380	#创建6380实例的数据目录
```

##4、启动Redis

```bash
[root@redis-master ~]# systemctl start redis
[root@redis-master ~]# redis-server /usr/local/redis/redis_6380.conf 
[root@redis-master ~]# ps -ef |grep redis
root      3536     1  0 09:33 ?        00:00:00 /usr/local/redis/src/redis-server 192.168.56.11:6379 [cluster]
root      3543     1  0 09:33 ?        00:00:00 redis-server 192.168.56.11:6380 [cluster]

[root@redis-slave01 ~]# systemctl start redis
[root@redis-slave01 ~]# redis-server /usr/local/redis/redis_6380.conf 
[root@redis-slave01 ~]# ps axu |grep redis
root      3821  0.5  0.7 153832  7692 ?        Ssl  09:35   0:00 /usr/local/redis/src/redis-server 192.168.56.12:6379 [cluster]
root      3826  0.5  0.6 153832  6896 ?        Ssl  09:35   0:00 redis-server 192.168.56.12:6380 [cluster]

[root@redis-slave02 ~]# systemctl start redis
[root@redis-slave02 ~]# redis-server /usr/local/redis/redis_6380.conf 
[root@redis-slave02 ~]# ps axu |grep redis
root      3801  0.7  0.7 153832  7696 ?        Ssl  09:36   0:00 /usr/local/redis/src/redis-server 192.168.56.13:6379 [cluster]
root      3806  1.4  0.7 153832  7692 ?        Ssl  09:36   0:00 redis-server 192.168.56.13:6380 [cluster]

```

## 5、部署Reids Cluster

如果虚拟机上开启了firewalld，所有机器需要增加如下规则，简单粗暴的方式是直接`systemctl stop firewalld`

```bash
firewall-cmd --permanent --add-port 6379-6380/tcp
firewall-cmd --permanent --add-port  16379-16380/tcp
firewall-cmd --reload
```

当前已经启动了6个节点的Redis服务：
`192.168.56.11:6379`
`192.168.56.11:6380`
`192.168.56.12:6379`
`192.168.56.12:6380`
`192.168.56.13:6379`
`192.168.56.13:6380`

下面在任一节点上执行以下构建集群的命令，将这里面的6个节点组建集群模式，`--cluster-replicas 1`表示每个主对应一个从。


```bash
[root@redis-master ~]# redis-cli --cluster create 192.168.56.11:6379 192.168.56.11:6380 192.168.56.12:6379 192.168.56.12:6380 192.168.56.13:6379 192.168.56.13:6380 --cluster-replicas 1
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460		#槽位的分配
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 192.168.56.12:6380 to 192.168.56.11:6379		#一主对应一从
Adding replica 192.168.56.11:6380 to 192.168.56.12:6379
Adding replica 192.168.56.13:6380 to 192.168.56.13:6379
>>> Trying to optimize slaves allocation for anti-affinity
[OK] Perfect anti-affinity obtained!
M: 31886d2098fb1e627bd71b5af000957a1e252787 192.168.56.11:6379
   slots:[0-5460] (5461 slots) master
S: be0ef4a1b1a60cee781afe5c2b8b5cbd7b68b4e6 192.168.56.11:6380
   replicates 8cd40e6a31c12e0b9c01f20056b9ecaa4db51446
M: 8cd40e6a31c12e0b9c01f20056b9ecaa4db51446 192.168.56.12:6379
   slots:[5461-10922] (5462 slots) master
S: eb812dc6051776e151bf69cd328bd0a66a20de01 192.168.56.12:6380
   replicates 587adfa041d0c0a14aa1a875bdec219a56b10201
M: 587adfa041d0c0a14aa1a875bdec219a56b10201 192.168.56.13:6379
   slots:[10923-16383] (5461 slots) master
S: 55a6f654dcb87c6a8017c8619f0ce8763a92abd6 192.168.56.13:6380
   replicates 31886d2098fb1e627bd71b5af000957a1e252787
Can I set the above configuration? (type 'yes' to accept): yes	#是否接受这样的配置，填yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
...
>>> Performing Cluster Check (using node 192.168.56.11:6379)
M: 31886d2098fb1e627bd71b5af000957a1e252787 192.168.56.11:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: 55a6f654dcb87c6a8017c8619f0ce8763a92abd6 192.168.56.13:6380
   slots: (0 slots) slave
   replicates 31886d2098fb1e627bd71b5af000957a1e252787
M: 8cd40e6a31c12e0b9c01f20056b9ecaa4db51446 192.168.56.12:6379
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: be0ef4a1b1a60cee781afe5c2b8b5cbd7b68b4e6 192.168.56.11:6380
   slots: (0 slots) slave
   replicates 8cd40e6a31c12e0b9c01f20056b9ecaa4db51446
S: eb812dc6051776e151bf69cd328bd0a66a20de01 192.168.56.12:6380
   slots: (0 slots) slave
   replicates 587adfa041d0c0a14aa1a875bdec219a56b10201
M: 587adfa041d0c0a14aa1a875bdec219a56b10201 192.168.56.13:6379
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
到此提示，集群创建成功！！！！！
```

## 6、连接集群

这里使用`redis-cli`去连接任一节点的Redis服务，可以进行一系列的日常操作，如下：

```bash
[root@redis-master ~]# redis-cli -c -h 192.168.56.11 -p 6380
192.168.56.11:6380> set k1 123	#k1键值存储定位在13这个节点上的6379实例
-> Redirected to slot [12706] located at 192.168.56.13:6379
OK
192.168.56.13:6379> set k2 abc	#k2键值存储定位在11这个节点上的6379实例
-> Redirected to slot [449] located at 192.168.56.11:6379
OK
192.168.56.11:6379> set k3 efg	#k3键值存储在本节点的实例上
OK
192.168.56.11:6379> KEYS *	#同样可以获取键值数据来查看数据的存储位置
1) "k2"
2) "k3"
192.168.56.11:6379> get k1
-> Redirected to slot [12706] located at 192.168.56.13:6379
"123"
192.168.56.13:6379> get k2
-> Redirected to slot [449] located at 192.168.56.11:6379
"abc"
192.168.56.11:6379> get k3
"efg"
```

## 7、管理集群

### 7.1、查看集群状态信息

```bash
[root@redis-master ~]# redis-cli --cluster check 192.168.56.11:6379
192.168.56.11:6379 (31886d20...) -> 2 keys | 5461 slots | 1 slaves.
192.168.56.12:6379 (8cd40e6a...) -> 0 keys | 5462 slots | 1 slaves.
192.168.56.13:6379 (587adfa0...) -> 1 keys | 5461 slots | 1 slaves.
[OK] 3 keys in 3 masters.
0.00 keys per slot on average.
>>> Performing Cluster Check (using node 192.168.56.11:6379)
M: 31886d2098fb1e627bd71b5af000957a1e252787 192.168.56.11:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: 55a6f654dcb87c6a8017c8619f0ce8763a92abd6 192.168.56.13:6380
   slots: (0 slots) slave
   replicates 31886d2098fb1e627bd71b5af000957a1e252787
M: 8cd40e6a31c12e0b9c01f20056b9ecaa4db51446 192.168.56.12:6379
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: be0ef4a1b1a60cee781afe5c2b8b5cbd7b68b4e6 192.168.56.11:6380
   slots: (0 slots) slave
   replicates 8cd40e6a31c12e0b9c01f20056b9ecaa4db51446
S: eb812dc6051776e151bf69cd328bd0a66a20de01 192.168.56.12:6380
   slots: (0 slots) slave
   replicates 587adfa041d0c0a14aa1a875bdec219a56b10201
M: 587adfa041d0c0a14aa1a875bdec219a56b10201 192.168.56.13:6379
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```
###7.2、添加集群节点

添加集群节点，在redis-master节点上，复制配置文件，增加6381和6382端口的实例，如下：

```bash
[root@redis-master redis]# cp redis_6380.conf redis_6381.conf
[root@redis-master redis]# cp redis_6380.conf redis_6382.conf
[root@redis-master redis]# vim redis_6381.conf 	#修改6381实例配置文件
Port 6381
pidfile "/var/run/redis_6381.pid"
logfile "/var/log/redis_6381.log"
dir "/var/redis_6381"

[root@redis-master redis]# vim redis_6382.conf 		#修改6382实例配置文件
Port 6382
pidfile "/var/run/redis_6382.pid"
logfile "/var/log/redis_6382.log"
dir "/var/redis_6382"

[root@redis-master redis]# mkdir /var/redis_6381	#创建新增实例的数据目录
[root@redis-master redis]# mkdir /var/redis_6382

[root@redis-master ~]# redis-server /usr/local/redis/redis_6381.conf 	#启动6381实例
[root@redis-master ~]# redis-server /usr/local/redis/redis_6382.conf 	#启动6382实例
[root@redis-master ~]# ps axu |grep redis
root      3536  0.3  0.7 156392  7712 ?        Ssl  09:34   1:07 /usr/local/redis/src/redis-server 192.168.56.11:6379 [cluster]
root      3543  0.3  0.3 169192  3388 ?        Ssl  09:34   1:21 redis-server 192.168.56.11:6380 [cluster]
root      4189  0.2  0.2 153832  2852 ?        Ssl  15:29   0:00 redis-server 192.168.56.11:6381 [cluster]
root      4194  0.1  0.2 153832  2852 ?        Ssl  15:29   0:00 redis-server 192.168.56.11:6382 [cluster]

```

这里添加集群节点有几种方式，可以将节点添加为主，也可以添加节点为从节点，也可以为节点指定给某个master节点作为从节点，下面是三种方式的不同增加方法，如下：

```bash
#将192.168.56.11:6381节点增加为集群的主节点，命令如下：
[root@redis-master ~]# redis-cli --cluster add-node 192.168.56.11:6381 192.168.56.11:6379
>>> Adding node 192.168.56.11:6381 to cluster 192.168.56.11:6379
>>> Performing Cluster Check (using node 192.168.56.11:6379)
M: 31886d2098fb1e627bd71b5af000957a1e252787 192.168.56.11:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: 55a6f654dcb87c6a8017c8619f0ce8763a92abd6 192.168.56.13:6380
   slots: (0 slots) slave
   replicates 31886d2098fb1e627bd71b5af000957a1e252787
M: 8cd40e6a31c12e0b9c01f20056b9ecaa4db51446 192.168.56.12:6379
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: be0ef4a1b1a60cee781afe5c2b8b5cbd7b68b4e6 192.168.56.11:6380
   slots: (0 slots) slave
   replicates 8cd40e6a31c12e0b9c01f20056b9ecaa4db51446
S: eb812dc6051776e151bf69cd328bd0a66a20de01 192.168.56.12:6380
   slots: (0 slots) slave
   replicates 587adfa041d0c0a14aa1a875bdec219a56b10201
M: 587adfa041d0c0a14aa1a875bdec219a56b10201 192.168.56.13:6379
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
>>> Send CLUSTER MEET to node 192.168.56.11:6381 to make it join the cluster.
[OK] New node added correctly.

#检查集群状态，可以看到192.168.56.11:6381已经添加为集群中的master
[root@redis-master ~]# redis-cli --cluster check 192.168.56.11:6379
192.168.56.11:6379 (31886d20...) -> 2 keys | 5461 slots | 1 slaves.
192.168.56.12:6379 (8cd40e6a...) -> 0 keys | 5462 slots | 1 slaves.
192.168.56.11:6381 (d04ed6a7...) -> 0 keys | 0 slots | 0 slaves.
192.168.56.13:6379 (587adfa0...) -> 1 keys | 5461 slots | 1 slaves.
[OK] 3 keys in 4 masters.
0.00 keys per slot on average.
>>> Performing Cluster Check (using node 192.168.56.11:6379)
M: 31886d2098fb1e627bd71b5af000957a1e252787 192.168.56.11:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: 55a6f654dcb87c6a8017c8619f0ce8763a92abd6 192.168.56.13:6380
   slots: (0 slots) slave
   replicates 31886d2098fb1e627bd71b5af000957a1e252787
M: 8cd40e6a31c12e0b9c01f20056b9ecaa4db51446 192.168.56.12:6379
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: be0ef4a1b1a60cee781afe5c2b8b5cbd7b68b4e6 192.168.56.11:6380
   slots: (0 slots) slave
   replicates 8cd40e6a31c12e0b9c01f20056b9ecaa4db51446
S: eb812dc6051776e151bf69cd328bd0a66a20de01 192.168.56.12:6380
   slots: (0 slots) slave
   replicates 587adfa041d0c0a14aa1a875bdec219a56b10201
M: d04ed6a7d3dbf0aaff0643b045fe22efc7c34500 192.168.56.11:6381
   slots: (0 slots) master
M: 587adfa041d0c0a14aa1a875bdec219a56b10201 192.168.56.13:6379
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

上面将`192.168.56.11:6381`实例添加为集群的master，现在将6382实例添加为集群的从节点，要知道的是，上面新增的master节点是没有从节点的，那么接下来新增的slave节点，会自动给master作为从节点，如下：

```bash
#将192.168.56.11:6382节点增加为集群的从节点，命令如下：
[root@redis-master ~]# redis-cli --cluster add-node 192.168.56.11:6382 192.168.56.11:6379 --cluster-slave
>>> Adding node 192.168.56.11:6382 to cluster 192.168.56.11:6379
>>> Performing Cluster Check (using node 192.168.56.11:6379)
M: 31886d2098fb1e627bd71b5af000957a1e252787 192.168.56.11:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: 55a6f654dcb87c6a8017c8619f0ce8763a92abd6 192.168.56.13:6380
   slots: (0 slots) slave
   replicates 31886d2098fb1e627bd71b5af000957a1e252787
M: 8cd40e6a31c12e0b9c01f20056b9ecaa4db51446 192.168.56.12:6379
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: be0ef4a1b1a60cee781afe5c2b8b5cbd7b68b4e6 192.168.56.11:6380
   slots: (0 slots) slave
   replicates 8cd40e6a31c12e0b9c01f20056b9ecaa4db51446
S: eb812dc6051776e151bf69cd328bd0a66a20de01 192.168.56.12:6380
   slots: (0 slots) slave
   replicates 587adfa041d0c0a14aa1a875bdec219a56b10201
M: d04ed6a7d3dbf0aaff0643b045fe22efc7c34500 192.168.56.11:6381
   slots: (0 slots) master
M: 587adfa041d0c0a14aa1a875bdec219a56b10201 192.168.56.13:6379
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
Automatically selected master 192.168.56.11:6381
>>> Send CLUSTER MEET to node 192.168.56.11:6382 to make it join the cluster.
Waiting for the cluster to join

>>> Configure node as replica of 192.168.56.11:6381.
[OK] New node added correctly.

#检查集群节点状态信息，可以看到6382的master节点id为：d04ed6a7d3dbf0aaff0643b045fe22efc7c34500，即6381实例。
[root@redis-master ~]# redis-cli --cluster check 192.168.56.11:6379
192.168.56.11:6379 (31886d20...) -> 2 keys | 5461 slots | 1 slaves.
192.168.56.12:6379 (8cd40e6a...) -> 0 keys | 5462 slots | 1 slaves.
192.168.56.11:6381 (d04ed6a7...) -> 0 keys | 0 slots | 1 slaves.
192.168.56.13:6379 (587adfa0...) -> 1 keys | 5461 slots | 1 slaves.
[OK] 3 keys in 4 masters.
0.00 keys per slot on average.
>>> Performing Cluster Check (using node 192.168.56.11:6379)
M: 31886d2098fb1e627bd71b5af000957a1e252787 192.168.56.11:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: 55a6f654dcb87c6a8017c8619f0ce8763a92abd6 192.168.56.13:6380
   slots: (0 slots) slave
   replicates 31886d2098fb1e627bd71b5af000957a1e252787
M: 8cd40e6a31c12e0b9c01f20056b9ecaa4db51446 192.168.56.12:6379
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: 91c803049aa0a12678e26521959aba25d4b06913 192.168.56.11:6382
   slots: (0 slots) slave
   replicates d04ed6a7d3dbf0aaff0643b045fe22efc7c34500
S: be0ef4a1b1a60cee781afe5c2b8b5cbd7b68b4e6 192.168.56.11:6380
   slots: (0 slots) slave
   replicates 8cd40e6a31c12e0b9c01f20056b9ecaa4db51446
S: eb812dc6051776e151bf69cd328bd0a66a20de01 192.168.56.12:6380
   slots: (0 slots) slave
   replicates 587adfa041d0c0a14aa1a875bdec219a56b10201
M: d04ed6a7d3dbf0aaff0643b045fe22efc7c34500 192.168.56.11:6381
   slots: (0 slots) master
   1 additional replica(s)
M: 587adfa041d0c0a14aa1a875bdec219a56b10201 192.168.56.13:6379
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

同样的，也可以为指定的master节点添加从节点，如将6382实例指定添加为6381实例的从节点：

```bash
redis-cli --cluster add-node 192.168.56.11:6382 192.168.56.11:6379 --cluster-slave --cluster-master-id d04ed6a7d3dbf0aaff0643b045fe22efc7c34500
```

### 7.3、分配slot

从上面的操作上，我们可以看到新增的master  6381实例上的slot为0，那么这里我们先给这个master进行分配一些slot，需要注意的是，reshard后面跟的ip，可以是集群中任一master ip。

```bash
[root@redis-master ~]# redis-cli --cluster reshard 192.168.56.11:6379
>>> Performing Cluster Check (using node 192.168.56.11:6379)
M: 31886d2098fb1e627bd71b5af000957a1e252787 192.168.56.11:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: 55a6f654dcb87c6a8017c8619f0ce8763a92abd6 192.168.56.13:6380
   slots: (0 slots) slave
   replicates 31886d2098fb1e627bd71b5af000957a1e252787
M: 8cd40e6a31c12e0b9c01f20056b9ecaa4db51446 192.168.56.12:6379
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: 91c803049aa0a12678e26521959aba25d4b06913 192.168.56.11:6382
   slots: (0 slots) slave
   replicates d04ed6a7d3dbf0aaff0643b045fe22efc7c34500
S: be0ef4a1b1a60cee781afe5c2b8b5cbd7b68b4e6 192.168.56.11:6380
   slots: (0 slots) slave
   replicates 8cd40e6a31c12e0b9c01f20056b9ecaa4db51446
S: eb812dc6051776e151bf69cd328bd0a66a20de01 192.168.56.12:6380
   slots: (0 slots) slave
   replicates 587adfa041d0c0a14aa1a875bdec219a56b10201
M: d04ed6a7d3dbf0aaff0643b045fe22efc7c34500 192.168.56.11:6381
   slots: (0 slots) master
   1 additional replica(s)
M: 587adfa041d0c0a14aa1a875bdec219a56b10201 192.168.56.13:6379
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
How many slots do you want to move (from 1 to 16384)? 100	#定义要分配多少slot，这里分配100个
What is the receiving node ID? d04ed6a7d3dbf0aaff0643b045fe22efc7c34500	#定义接收slot的node id，即新的master id
Please enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1: 31886d2098fb1e627bd71b5af000957a1e252787	#定义第一个源master的id，如果想在所有master上拿slot，直接敲all
Source node #2: 587adfa041d0c0a14aa1a875bdec219a56b10201	##定义第二个源master的id，如果不再继续有新的源，直接敲done
Source node #3: done
Ready to move 100 slots.
  Source nodes:
    M: 31886d2098fb1e627bd71b5af000957a1e252787 192.168.56.11:6379
       slots:[0-5460] (5461 slots) master
       1 additional replica(s)
    M: 587adfa041d0c0a14aa1a875bdec219a56b10201 192.168.56.13:6379
       slots:[10923-16383] (5461 slots) master
       1 additional replica(s)
  Destination node:
    M: d04ed6a7d3dbf0aaff0643b045fe22efc7c34500 192.168.56.11:6381
       slots: (0 slots) master
       1 additional replica(s)
  Resharding plan:
    Moving slot 0 from 31886d2098fb1e627bd71b5af000957a1e252787
    Moving slot 1 from 31886d2098fb1e627bd71b5af000957a1e252787
	......
Do you want to proceed with the proposed reshard plan (yes/no)? yes
Moving slot 0 from 192.168.56.11:6379 to 192.168.56.11:6381: 
Moving slot 1 from 192.168.56.11:6379 to 192.168.56.11:6381: 
Moving slot 2 from 192.168.56.11:6379 to 192.168.56.11:6381: 
Moving slot 3 from 192.168.56.11:6379 to 192.168.56.11:6381: 
......

#重新查看集群中的slot，可以看到新的master上已经分配了100个slot
[root@redis-master ~]# redis-cli --cluster check 192.168.56.11:6379	
......
M: d04ed6a7d3dbf0aaff0643b045fe22efc7c34500 192.168.56.11:6381
   slots:[0-49],[10923-10972] (100 slots) master
   1 additional replica(s)
......
```

slot的分配，也可以作为slot的迁移，在当前某个master节点出现故障时，我们可以通过slot的分配对故障节点的slot进行迁移，命令和分配的命令是一样的，如下：

```bash
redis-cli --cluster reshard 192.168.56.11:6379
```

### 7.4、删除集群节点

有添加，就有删除，删除集群中的节点，本身节点上的Redis必须不带有slot，如果有slot，需要先进行移除，否则会报错。被删除的node重启后，依然会记得集群中的其他节点，这就需要执行`cluster forget nodeid`来忘记其他节点。

```bash
#这里先删除6382这个从节点实例，因为该实例上没有slot，是可以直接移除的
[root@redis-master ~]# redis-cli --cluster del-node 192.168.56.11:6382 91c803049aa0a12678e26521959aba25d4b06913
>>> Removing node 91c803049aa0a12678e26521959aba25d4b06913 from cluster 192.168.56.11:6382
>>> Sending CLUSTER FORGET messages to the cluster...
>>> SHUTDOWN the node.

[root@redis-master ~]# redis-cli --cluster check 192.168.56.11:6379	#重新查看是已经没有了6382实例了
192.168.56.11:6379 (31886d20...) -> 2 keys | 5411 slots | 1 slaves.
192.168.56.12:6379 (8cd40e6a...) -> 0 keys | 5462 slots | 1 slaves.
192.168.56.11:6381 (d04ed6a7...) -> 0 keys | 100 slots | 0 slaves.
192.168.56.13:6379 (587adfa0...) -> 1 keys | 5411 slots | 1 slaves.
[OK] 3 keys in 4 masters.
0.00 keys per slot on average.
>>> Performing Cluster Check (using node 192.168.56.11:6379)
M: 31886d2098fb1e627bd71b5af000957a1e252787 192.168.56.11:6379
   slots:[50-5460] (5411 slots) master
   1 additional replica(s)
S: 55a6f654dcb87c6a8017c8619f0ce8763a92abd6 192.168.56.13:6380
   slots: (0 slots) slave
   replicates 31886d2098fb1e627bd71b5af000957a1e252787
M: 8cd40e6a31c12e0b9c01f20056b9ecaa4db51446 192.168.56.12:6379
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: be0ef4a1b1a60cee781afe5c2b8b5cbd7b68b4e6 192.168.56.11:6380
   slots: (0 slots) slave
   replicates 8cd40e6a31c12e0b9c01f20056b9ecaa4db51446
S: eb812dc6051776e151bf69cd328bd0a66a20de01 192.168.56.12:6380
   slots: (0 slots) slave
   replicates 587adfa041d0c0a14aa1a875bdec219a56b10201
M: d04ed6a7d3dbf0aaff0643b045fe22efc7c34500 192.168.56.11:6381
   slots:[0-49],[10923-10972] (100 slots) master
M: 587adfa041d0c0a14aa1a875bdec219a56b10201 192.168.56.13:6379
   slots:[10973-16383] (5411 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.

#那么现在再来删除带有slot的6381实例，则会出现报错，如下
[root@redis-master ~]# redis-cli --cluster del-node 192.168.56.11:6381 d04ed6a7d3dbf0aaff0643b045fe22efc7c34500
>>> Removing node d04ed6a7d3dbf0aaff0643b045fe22efc7c34500 from cluster 192.168.56.11:6381
[ERR] Node 192.168.56.11:6381 is not empty! Reshard data away and try again.

#提示该实例不为空，需要Reshard掉数据后再重试，那么现在先将slot进行迁移掉

[root@redis-master ~]# redis-cli --cluster reshard 192.168.56.11:6381
>>> Performing Cluster Check (using node 192.168.56.11:6381)
M: d04ed6a7d3dbf0aaff0643b045fe22efc7c34500 192.168.56.11:6381
   slots:[0-49],[10923-10972] (100 slots) master
......
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
How many slots do you want to move (from 1 to 16384)? 100
What is the receiving node ID? 587adfa041d0c0a14aa1a875bdec219a56b10201
Please enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1: d04ed6a7d3dbf0aaff0643b045fe22efc7c34500
Source node #2: done

[root@redis-master ~]# redis-cli --cluster check 192.168.56.11:6379
192.168.56.11:6379 (31886d20...) -> 2 keys | 5411 slots | 1 slaves.
192.168.56.12:6379 (8cd40e6a...) -> 0 keys | 5462 slots | 1 slaves.
192.168.56.11:6381 (d04ed6a7...) -> 0 keys | 0 slots | 0 slaves.	#slots数量已经为0
192.168.56.13:6379 (587adfa0...) -> 1 keys | 5511 slots | 1 slaves.
......

#再次进行删除6381集群节点，即可顺利完成删除
[root@redis-master ~]# redis-cli --cluster del-node 192.168.56.11:6381 d04ed6a7d3dbf0aaff0643b045fe22efc7c34500
>>> Removing node d04ed6a7d3dbf0aaff0643b045fe22efc7c34500 from cluster 192.168.56.11:6381
>>> Sending CLUSTER FORGET messages to the cluster...
>>> SHUTDOWN the node.
```

### 7.5、slot数量平衡

先通过reshard功能，模拟slot不均匀分配，然后检查集群状态中，可以看到各个master上的slot数量不均衡，如下：

```bash
[root@redis-master ~]# redis-cli --cluster check 192.168.56.11:6379
192.168.56.11:6379 (31886d20...) -> 1 keys | 4416 slots | 1 slaves.
192.168.56.12:6379 (8cd40e6a...) -> 0 keys | 4457 slots | 1 slaves.
192.168.56.13:6379 (587adfa0...) -> 2 keys | 7511 slots | 1 slaves.
......
```

当我们使用的slot数量分布不均匀时，同样也可以使用平衡功能，将各个节点的slot进行重新均衡分配：

```bash
[root@redis-master ~]# redis-cli --cluster rebalance --cluster-threshold 1 192.168.56.11:6379
>>> Performing Cluster Check (using node 192.168.56.11:6379)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
>>> Rebalancing across 3 nodes. Total weight = 3.00
Moving 1046 slots from 192.168.56.13:6379 to 192.168.56.11:6379
###################################################################################################
Moving 1004 slots from 192.168.56.13:6379 to 192.168.56.12:6379
###################################################################################################

[root@redis-master ~]# redis-cli --cluster check 192.168.56.11:6379
192.168.56.11:6379 (31886d20...) -> 2 keys | 5462 slots | 1 slaves.
192.168.56.12:6379 (8cd40e6a...) -> 0 keys | 5461 slots | 1 slaves.
192.168.56.13:6379 (587adfa0...) -> 1 keys | 5461 slots | 1 slaves.
......

#需要注意的是，所有节点的平衡阈值在1%，当各个节点的slot数值相差不到1%时，进行balance是会报以下提示的：
*** No rebalancing needed! All nodes are within the 1.00% threshold.
```

### 7.3、集群外部数据导入

先解除6381实例的集群模式，重新启动6381实例，并写入数据，如下

```bash
[root@redis-master ~]# redis-cli -h 192.168.56.11 -p 6381
192.168.56.11:6381> KEYS *
(empty list or set)
192.168.56.11:6381> set k0 aaa
OK
192.168.56.11:6381> set k5 bbb
OK
192.168.56.11:6381> exit

```

再将6381实例的数据导入到集群中，如下

```bash
[root@redis-master ~]# redis-cli --cluster import 192.168.56.11:6379 --cluster-from 192.168.56.11:6381 --cluster-copy
>>> Importing data from 192.168.56.11:6381 to cluster 192.168.56.11:6379
>>> Performing Cluster Check (using node 192.168.56.11:6379)
M: 31886d2098fb1e627bd71b5af000957a1e252787 192.168.56.11:6379
   slots:[0-5461] (5462 slots) master
   1 additional replica(s)
S: 55a6f654dcb87c6a8017c8619f0ce8763a92abd6 192.168.56.13:6380
   slots: (0 slots) slave
   replicates 31886d2098fb1e627bd71b5af000957a1e252787
M: 8cd40e6a31c12e0b9c01f20056b9ecaa4db51446 192.168.56.12:6379
   slots:[5462-10922] (5461 slots) master
   1 additional replica(s)
S: be0ef4a1b1a60cee781afe5c2b8b5cbd7b68b4e6 192.168.56.11:6380
   slots: (0 slots) slave
   replicates 8cd40e6a31c12e0b9c01f20056b9ecaa4db51446
S: eb812dc6051776e151bf69cd328bd0a66a20de01 192.168.56.12:6380
   slots: (0 slots) slave
   replicates 587adfa041d0c0a14aa1a875bdec219a56b10201
M: 587adfa041d0c0a14aa1a875bdec219a56b10201 192.168.56.13:6379
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
*** Importing 2 keys from DB 0
Migrating k5 to 192.168.56.13:6379: OK
Migrating k0 to 192.168.56.12:6379: OK

[root@redis-master ~]# redis-cli -c -h 192.168.56.11 -p 6379	#登录集群查看数据
192.168.56.11:6379> KEYS *
1) "k2"
2) "k3"
192.168.56.11:6379> get k0
-> Redirected to slot [8579] located at 192.168.56.12:6379
"aaa"
192.168.56.12:6379> get k5
-> Redirected to slot [12582] located at 192.168.56.13:6379
"bbb"
```

这里需要注意的是：`Cluster-from`后面跟外部`redis的ip和port `如果只使用`cluster-copy`，则要导入集群中的key不能在，否则如下： 如果集群中已有同样的key，如果需要替换，可以`cluster-copy和cluster-replace`联用，这样集群中的`key`就会被替换为外部的。







生产环境中，为了保证Redis的高可用性，通常会配置一主两从的模式，也就是说`--cluster-replicas 2`







