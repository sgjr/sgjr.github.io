---
layout:       post
title:        "redis-cluster部署及数据迁移"
subtitle:     "redis-cluster"
header-mask:  0.3
catalog:      true
multilingual: true
tags:
    - redis

---


## 工作原理
节选自redis官方文档：http://www.redis.cn/topics/cluster-tutorial.html
### Redis集群介绍
Redis 集群是一个提供在多个Redis间节点间共享数据的程序集。
Redis集群并不支持处理多个keys的命令,因为这需要在不同的节点间移动数据,从而达不到像Redis那样的性能,在高负载的情况下可能会导致不可预料的错误.
Redis 集群通过分区来提供一定程度的可用性,在实际环境中当某个节点宕机或者不可达的情况下继续处理命令. Redis 集群的优势:
- 自动分割数据到不同的节点上。
- 整个集群的部分节点失败或者不可达的情况下能够继续处理命令。
### Redis 集群的数据分片
Redis 集群没有使用一致性hash, 而是引入了 哈希槽的概念.
Redis 集群有16384个哈希槽,每个key通过CRC16校验后对16384取模来决定放置哪个槽.集群的每个节点负责一部分hash槽,举个例子,比如当前集群有3个节点,那么:
- 节点 A 包含 0 到 5500号哈希槽.
- 节点 B 包含5501 到 11000 号哈希槽.
- 节点 C 包含11001 到 16384号哈希槽.

这种结构很容易添加或者删除节点. 比如如果我想新添加个节点D, 我需要从节点 A, B, C中得部分槽到D上. 如果我像移除节点A,需要将A中得槽移到B和C节点上,然后将没有任何槽的A节点从集群中移除即可. 由于从一个节点将哈希槽移动到另一个节点并不会停止服务,所以无论添加删除或者改变某个节点的哈希槽的数量都不会造成集群不可用的状态.
### Redis 集群的主从复制模型
为了使在部分节点失败或者大部分节点无法通信的情况下集群仍然可用，所以集群使用了主从复制模型,每个节点都会有N-1个复制品.
在我们例子中具有A，B，C三个节点的集群,在没有复制模型的情况下,如果节点B失败了，那么整个集群就会以为缺少5501-11000这个范围的槽而不可用.
然而如果在集群创建的时候（或者过一段时间）我们为每个节点添加一个从节点A1，B1，C1,那么整个集群便有三个master节点和三个slave节点组成，这样在节点B失败后，集群便会选举B1为新的主节点继续服务，整个集群便不会因为槽找不到而不可用了
不过当B和B1 都失败后，集群是不可用的.


---
## redis安装过程
1、下载redis安装包

2、解压
```
cd /app
tar zxf redis-3.2.11.tar.gz 
mv redis-3.2.11.tar.gz redis
```

3、编译安装
```
cd redis

make && make install

```

4、创建相关目录,将相关命令从src目录复制到bin目录
```
mkdir -pv /app/redis/{bin,conf,data,logs}

```

5、使用utils中的install_server.sh安装redis server

---

## 部署过程
### Redis-cluster搭建
1、从测试打包已经编译redis文件夹，上传至需要安装的服务器

2、解压至app目录，目录结构如下，其中data,conf,logs为编译完成后自行创建的目录，分别用来存放数据文件，配置文件，日志文件，data目录下自行创建以端口号命名的文件夹，分别用来存放各个端口号实例的数据文件
```
-rw-rw-r--  1 root root 92766 Sep 21 22:20 00-RELEASENOTES
drwxr-xr-x  2 root root  4096 Dec 20 16:40 bin
-rw-rw-r--  1 root root    53 Sep 21 22:20 BUGS
drwxr-xr-x  2 root root  4096 Jan  4 15:37 conf
-rw-rw-r--  1 root root  1805 Sep 21 22:20 CONTRIBUTING
-rw-rw-r--  1 root root  1487 Sep 21 22:20 COPYING
drwxr-xr-x 10 root root  4096 Jan  3 11:24 data
drwxrwxr-x  7 root root  4096 Dec 20 14:50 deps
-rw-rw-r--  1 root root    11 Sep 21 22:20 INSTALL
drwxr-xr-x  2 root root  4096 Jan  4 15:37 logs
-rw-rw-r--  1 root root   151 Sep 21 22:20 Makefile
-rw-rw-r--  1 root root  4223 Sep 21 22:20 MANIFESTO
-rw-rw-r--  1 root root  6834 Sep 21 22:20 README.md
-rw-rw-r--  1 root root 46695 Sep 21 22:20 redis.conf
-rwxrwxr-x  1 root root   271 Sep 21 22:20 runtest
-rwxrwxr-x  1 root root   280 Sep 21 22:20 runtest-cluster
-rwxrwxr-x  1 root root   281 Sep 21 22:20 runtest-sentinel
-rw-rw-r--  1 root root  7606 Sep 21 22:20 sentinel.conf
drwxrwxr-x  2 root root  4096 Dec 20 14:51 src
drwxrwxr-x 10 root root  4096 Sep 21 22:20 tests
drwxrwxr-x  7 root root  4096 Jan  4 14:36 utils
```
3、修改各个端口的配置文件，以node1:7000的配置文件为例，根据情况修改
```
bind node1
protected-mode yes
port 7000
tcp-backlog 511
timeout 0
tcp-keepalive 300
daemonize yes
supervised no
pidfile /var/run/redis_7000.pid
loglevel notice
logfile /app/redis/logs/redis_7000.log
databases 16
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir /app/redis/data/7000
slave-serve-stale-data yes
slave-read-only yes
repl-diskless-sync no
repl-diskless-sync-delay 5
repl-disable-tcp-nodelay no
slave-priority 100
appendonly no
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-load-truncated yes
lua-time-limit 5000
cluster-enabled yes
cluster-config-file nodes-7000.conf
cluster-node-timeout 15000
slowlog-log-slower-than 10000
slowlog-max-len 128
latency-monitor-threshold 0
notify-keyspace-events ""
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10
aof-rewrite-incremental-fsync yes
```
4、启动对应端口的服务（所有节点的主从都启动）,以7000为例
```
/app/redis/bin/redis-server  /app/redis/conf/7000.conf
```
5、创建集群
```
 /app/redis/bin/redis-trib.rb create node1:7000 node2:7000 node2:7000

[root@node1 bin]# ./redis-trib.rb check node1:7000
>>> Performing Cluster Check (using node node1:7000)
M: 3b485951a72e16133464585dc2920c10ce75967b node1:7000
   slots:5461-10922 (5461 slots) master
   0 additional replica(s)
M: 54d5eb9415289228b05b28fe585bd5ab6e9bc0bc node3:7000
   slots:0-5460 (5461 slots) master
   0 additional replica(s)
M: b6dd7c3d5af4ffc37e70d7eb538a3f9b0ae5a84f node2:7000
   slots:10923-16383 (5461 slots) master
   0 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```
6、从节点的添加需要在数据迁移之后

### 数据迁移
根据实际情况，参考：https://www.18188.org/articles/2016/04/23/1461374145366.html 进行数据迁移。

1、将所有的slot都转移到一台主实例
```

[root@node1 bin]# ./redis-trib.rb reshard --from b6dd7c3d5af4ffc37e70d7eb538a3f9b0ae5a84f   --to 3b485951a72e16133464585dc2920c10ce75967b  --slots 5462 --yes node1:7000

[root@node1 bin]# ./redis-trib.rb reshard --from 54d5eb9415289228b05b28fe585bd5ab6e9bc0bc   --to 3b485951a72e16133464585dc2920c10ce75967b  --slots 5462 --yes node1:7000
```
2、将原有的阿里云redis备份rdb文件下载放到对应目录
```
cp /app/hins2533955_data_20180105195042.rdb /app/redis/data/7000/dump.rdb
```

3、重启将备份文件读入,重启完成连接进入使用dbsize查看数据是否导入
```
/app/redis/bin/redis-cli -h node1 -p 7000 shutdown
/app/redis/bin/redis-server /app/redis/conf/7000.conf

```

4、重新分配slot
```
[root@node1 bin]# ./redis-trib.rb reshard --from 3b485951a72e16133464585dc2920c10ce75967b   --to b6dd7c3d5af4ffc37e70d7eb538a3f9b0ae5a84f  --slots 5462 --yes node2:7000


[root@node1 bin]# ./redis-trib.rb reshard --from 3b485951a72e16133464585dc2920c10ce75967b   --to 54d5eb9415289228b05b28fe585bd5ab6e9bc0bc  --slots 5462 --yes node3:7000

```
5、增加从节点,各节点对应关系：
```
1.node1:7000(主)---》node4:7000(从)
2.node2:7000(主)---》node5:7000(从)
3.node3:7000(主)---》 node6:7000(从)
```
```
[root@node1 bin]# ./redis-trib.rb check node1.7000
Invalid IP or Port (given as node1.7000) - use IP:Port format
[root@node1 bin]# ./redis-trib.rb check node1:7000
>>> Performing Cluster Check (using node node1:7000)
M: 3b485951a72e16133464585dc2920c10ce75967b node1:7000
   slots:10924-16383 (5460 slots) master
   0 additional replica(s)
M: b6dd7c3d5af4ffc37e70d7eb538a3f9b0ae5a84f node2:7000
   slots:0-5461 (5462 slots) master
   0 additional replica(s)
M: 54d5eb9415289228b05b28fe585bd5ab6e9bc0bc node3:7000
   slots:5462-10923 (5462 slots) master
   0 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
[root@node1 bin]# ./redis-trib.rb add-node --slave --master-id 3b485951a72e16133464585dc2920c10ce75967b  node4:7000 node1:7000
>>> Adding node node4:7000 to cluster node1:7000
>>> Performing Cluster Check (using node node1:7000)
M: 3b485951a72e16133464585dc2920c10ce75967b node1:7000
   slots:10924-16383 (5460 slots) master
   0 additional replica(s)
M: b6dd7c3d5af4ffc37e70d7eb538a3f9b0ae5a84f node2:7000
   slots:0-5461 (5462 slots) master
   0 additional replica(s)
M: 54d5eb9415289228b05b28fe585bd5ab6e9bc0bc node3:7000
   slots:5462-10923 (5462 slots) master
   0 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
>>> Send CLUSTER MEET to node node4:7000 to make it join the cluster.
Waiting for the cluster to join...
>>> Configure node as replica of node1:7000.
[OK] New node added correctly.
[root@node1 bin]# ./redis-trib.rb add-node --slave --master-id b6dd7c3d5af4ffc37e70d7eb538a3f9b0ae5a84f  node5:7000 node2:7000
>>> Adding node node5:7000 to cluster node2:7000
>>> Performing Cluster Check (using node node2:7000)
M: b6dd7c3d5af4ffc37e70d7eb538a3f9b0ae5a84f node2:7000
   slots:0-5461 (5462 slots) master
   0 additional replica(s)
S: 8ff59145b1d3d7fd57923f3cb9c3444f7236d7b1 node4:7000
   slots: (0 slots) slave
   replicates 3b485951a72e16133464585dc2920c10ce75967b
M: 54d5eb9415289228b05b28fe585bd5ab6e9bc0bc node3:7000
   slots:5462-10923 (5462 slots) master
   0 additional replica(s)
M: 3b485951a72e16133464585dc2920c10ce75967b node1:7000
   slots:10924-16383 (5460 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
>>> Send CLUSTER MEET to node node5:7000 to make it join the cluster.
Waiting for the cluster to join.
>>> Configure node as replica of node2:7000.
[OK] New node added correctly.
[root@node1 bin]# ./redis-trib.rb add-node --slave --master-id 54d5eb9415289228b05b28fe585bd5ab6e9bc0bc  node6:7000 node3:7000
>>> Adding node node6:7000 to cluster node3:7000
>>> Performing Cluster Check (using node node3:7000)
M: 54d5eb9415289228b05b28fe585bd5ab6e9bc0bc node3:7000
   slots:5462-10923 (5462 slots) master
   0 additional replica(s)
S: c09449b9dce5af4ac485ccedb664188229d75430 node5:7000
   slots: (0 slots) slave
   replicates b6dd7c3d5af4ffc37e70d7eb538a3f9b0ae5a84f
S: 8ff59145b1d3d7fd57923f3cb9c3444f7236d7b1 node4:7000
   slots: (0 slots) slave
   replicates 3b485951a72e16133464585dc2920c10ce75967b
M: b6dd7c3d5af4ffc37e70d7eb538a3f9b0ae5a84f node2:7000
   slots:0-5461 (5462 slots) master
   1 additional replica(s)
M: 3b485951a72e16133464585dc2920c10ce75967b node1:7000
   slots:10924-16383 (5460 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
>>> Send CLUSTER MEET to node node6:7000 to make it join the cluster.
Waiting for the cluster to join.
>>> Configure node as replica of node3:7000.
[OK] New node added correctly.


[root@node1 bin]# ./redis-trib.rb check node1:7000
>>> Performing Cluster Check (using node node1:7000)
M: 3b485951a72e16133464585dc2920c10ce75967b node1:7000
   slots:10924-16383 (5460 slots) master
   1 additional replica(s)
S: 8ff59145b1d3d7fd57923f3cb9c3444f7236d7b1 node4:7000
   slots: (0 slots) slave
   replicates 3b485951a72e16133464585dc2920c10ce75967b
S: 7501954b635724d66285afbb95841f4b49df5645 node6:7000
   slots: (0 slots) slave
   replicates 54d5eb9415289228b05b28fe585bd5ab6e9bc0bc
S: c09449b9dce5af4ac485ccedb664188229d75430 node5:7000
   slots: (0 slots) slave
   replicates b6dd7c3d5af4ffc37e70d7eb538a3f9b0ae5a84f
M: 54d5eb9415289228b05b28fe585bd5ab6e9bc0bc node3:7000
   slots:5462-10923 (5462 slots) master
   1 additional replica(s)
M: b6dd7c3d5af4ffc37e70d7eb538a3f9b0ae5a84f node2:7000
   slots:0-5461 (5462 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.


```
