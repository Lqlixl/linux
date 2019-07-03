# redis cluster
## redis cluster 实现
准备主机6台
|servername|ip|
|:-|:-|
|redis1|192.168.27.11|
|redis2|192.168.27.12|
|redis3|192.168.27.13|
|slave1|192.168.27.14|
|slave2|192.168.27.15|
|slave3|192.168.27.16|
## 配置集群的条件
1.确保每个redis节点使用相同的配置和密码
2.每个节点必须开启的参数
```bash
cluster-enabled yes #必须开启集群状态，开启后redis进程会有cluster显示 
cluster-config-file nodes-6380.conf #此文件有redis cluster集群自动创建和维护，不需要任何手动操作
```
3.所有的redis服务器必须没有任何数据
### 配置环境
1.修改配置文件
```bash
bind 0.0.0.0
daemonize yes
supervised systemd
logfile "/apps/redis/logs/redis.log"
save 5 1
stop-writes-on-bgsave-error no
dir /apps/redis/logs/
masterauth 111111
requirepass 111111
maxclients 65535
maxmemory 1024000000
cluster-enabled yes
cluster-config-file nodes-6379.conf
```
2.将配置文件发送至所有的redis服务器

```bash
[root@redis1 ~]# scp /apps/redis/etc/redis.conf 192.168.27.11:/apps/redis/etc/redis.conf
[root@redis1 ~]# scp /apps/redis/etc/redis.conf 192.168.27.12:/apps/redis/etc/redis.conf
[root@redis1 ~]# scp /apps/redis/etc/redis.conf 192.168.27.13:/apps/redis/etc/redis.conf
[root@redis1 ~]# scp /apps/redis/etc/redis.conf 192.168.27.14:/apps/redis/etc/redis.conf
[root@redis1 ~]# scp /apps/redis/etc/redis.conf 192.168.27.15:/apps/redis/etc/redis.conf
[root@redis1 ~]# scp /apps/redis/etc/redis.conf 192.168.27.16:/apps/redis/etc/redis.conf
```
3.在所有redis服务器上启动服务

```bash
[root@redis1 ~]# redis-server /apps/redis/etc/redis.conf 
```
4.查看所有主机上集群服务器的16379端口是否开启

```bash
[root@redis1 ~]# ss -tnl | grep 16379
LISTEN     0      511          *:16379                    *:*   
```
5.查看数据库确保没有数据
```bash
[root@redis1 ~]# redis-cli 
127.0.0.1:6379> auth 111111
OK
127.0.0.1:6379> KEYS *
(empty list or set)
#如果数据库内有数据集群将无法成功创建
```
以上基础环境准备完毕
### 创建集群
redis3和4版本在创建集群时需要使用到管理集群工具redis-trib.rb，这个工具是redis官方推出的管理redis集群工具，集成在redis的源码src目录下，是基于redis提供的集群命令封装成简单、便捷使用的操作工具，redis-trib.rb是redis作者用ruby开发完成的，centos系统yum安装的ruby存在版本较低的问题，所以需要编译安装ruby。
#### 编译安装ruby
1.解压源码包
```bash
[root@redis1 ~]# tar -xf ruby-2.5.5.tar.gz 
```
2.检查编译环境和编译所需要的组件
```bash
[root@redis1 ruby-2.5.5]# ./configure 
```
3.编译安装

```bash
[root@redis1 ruby-2.5.5]# make && make install
```
4.ruby编译完成后还需要安装一个redis的模块
```bash
[root@redis1 ruby-2.5.5]# gem install redis
```
以上安装完毕redis-trib.rb可以正常使用了
### 使用redis-trib.rb创建集群
1.先将源码包内的工具复制到redis安装目录下并创建软连接

```bash
[root@redis1 ~]# cp redis-4.0.14/src/redis-trib.rb /apps/redis/bin/
[root@redis1 ~]# ln -s /apps/redis/bin/* /usr/bin/
```
2.创建集群

```bash
[root@redis1 ~]# redis-trib.rb create --replicas 192.168.27.11:6379 192.168.27.12:6379 192.168.27.13:679 192.168.27.14:6379 192.168.27.15:6379 192.168.27.16:6379
>>> Creating cluster
[ERR] Sorry, can't connect to node 192.168.27.12:6379
#报错，这是由于没有将密码写入配置文件所导致的
```
3.在建立集群前需要将集群密码写入client.rb配置文件
```bash
[root@redis1 ~]# vim /usr/local/lib/ruby/gems/2.5.0/gems/redis-4.1.2/lib/redis/client.rb 
   DEFAULTS = {
      :url => lambda { ENV["REDIS_URL"] },
      :scheme => "redis",
      :host => "127.0.0.1",
      :port => 6379,
      :path => nil,
      :timeout => 5.0,
      :password => 111111,          #在此处将集群的密码写入，如果不写入将报错
      :db => 0,
      :driver => nil,
      :id => nil,
      :tcp_keepalive => 0,
      :reconnect_attempts => 1,
      :reconnect_delay => 0,
      :reconnect_delay_max => 0.5,
      :inherit_socket => false
    }
```
4.再次创建集群
```bash
[root@redis1 ~]# redis-trib.rb create --replicas 1 192.168.27.11:6379 192.168.27.12:6379 192.168.27.13:6379 192.168.27.14:6379 192.168.27.15:6379 192.168.27.16:6379
>>> Creating cluster
>>> Performing hash slots allocation on 6 nodes...
Using 3 masters:                                                        #创建了3个master
192.168.27.11:6379                                                          
192.168.27.12:6379
192.168.27.13:6379
Adding replica 192.168.27.15:6379 to 192.168.27.11:6379                 #将192.168.27.15添加为192.168.27.11的从
Adding replica 192.168.27.16:6379 to 192.168.27.12:6379                 #将192.168.27.16添加为192.168.27.12的从
Adding replica 192.168.27.14:6379 to 192.168.27.13:6379                 #将192.168.27.14添加为192.168.27.13的从
M: 58e21837e91adc28f55cd909e22d610ba4d5e18c 192.168.27.11:6379
   slots:0-5460 (5461 slots) master
M: 3376d3a9d73316075955fb59afa6e03b525419a9 192.168.27.12:6379
   slots:5461-10922 (5462 slots) master
M: 87d5d9278ed230d0817ac36e59dd2d39487e878c 192.168.27.13:6379
   slots:10923-16383 (5461 slots) master
S: b6406a84234193e958011b21b01c410aed019091 192.168.27.14:6379
   replicates 87d5d9278ed230d0817ac36e59dd2d39487e878c
S: 7641530f649de98c30f4f02cdc42c299f4c339e4 192.168.27.15:6379
   replicates 58e21837e91adc28f55cd909e22d610ba4d5e18c
S: cc3d1d0c9c64bbce0587507f5d4b0e3233691e2f 192.168.27.16:6379
   replicates 3376d3a9d73316075955fb59afa6e03b525419a9
# M表示master的标记位，S表示slave的标记位，后面所跟的为redis的id号
Can I set the above configuration? (type 'yes' to accept): yes          #是否生成配置文件
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join...
>>> Performing Cluster Check (using node 192.168.27.11:6379)
M: 58e21837e91adc28f55cd909e22d610ba4d5e18c 192.168.27.11:6379
   slots:0-5460 (5461 slots) master                                     #分配了5461个槽位
   1 additional replica(s)
M: 3376d3a9d73316075955fb59afa6e03b525419a9 192.168.27.12:6379
   slots:5461-10922 (5462 slots) master                                 #分配了5462个槽位
   1 additional replica(s)
S: b6406a84234193e958011b21b01c410aed019091 192.168.27.14:6379
   slots: (0 slots) slave
   replicates 87d5d9278ed230d0817ac36e59dd2d39487e878c
S: cc3d1d0c9c64bbce0587507f5d4b0e3233691e2f 192.168.27.16:6379
   slots: (0 slots) slave
   replicates 3376d3a9d73316075955fb59afa6e03b525419a9
S: 7641530f649de98c30f4f02cdc42c299f4c339e4 192.168.27.15:6379
   slots: (0 slots) slave
   replicates 58e21837e91adc28f55cd909e22d610ba4d5e18c
M: 87d5d9278ed230d0817ac36e59dd2d39487e878c 192.168.27.13:6379
   slots:10923-16383 (5461 slots) master                                #分配了5461个槽位
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.                                           #所有的16384个槽位分配完毕
```
### 验证
查看集群的详细状态
```bash
[root@redis1 ~]# redis-trib.rb check 192.168.27.11:6379
>>> Performing Cluster Check (using node 192.168.27.11:6379)
M: 58e21837e91adc28f55cd909e22d610ba4d5e18c 192.168.27.11:6379
   slots:0-5460 (5461 slots) master
   1 additional replica(s)
M: 3376d3a9d73316075955fb59afa6e03b525419a9 192.168.27.12:6379
   slots:5461-10922 (5462 slots) master
   1 additional replica(s)
S: b6406a84234193e958011b21b01c410aed019091 192.168.27.14:6379
   slots: (0 slots) slave
   replicates 87d5d9278ed230d0817ac36e59dd2d39487e878c
S: cc3d1d0c9c64bbce0587507f5d4b0e3233691e2f 192.168.27.16:6379
   slots: (0 slots) slave
   replicates 3376d3a9d73316075955fb59afa6e03b525419a9
S: 7641530f649de98c30f4f02cdc42c299f4c339e4 192.168.27.15:6379
   slots: (0 slots) slave
   replicates 58e21837e91adc28f55cd909e22d610ba4d5e18c
M: 87d5d9278ed230d0817ac36e59dd2d39487e878c 192.168.27.13:6379
   slots:10923-16383 (5461 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```
简洁的显示集群信息
```bash
[root@redis1 ~]# redis-trib.rb info 192.168.27.11:6379
192.168.27.11:6379 (58e21837...) -> 0 keys | 5461 slots | 1 slaves.
192.168.27.12:6379 (3376d3a9...) -> 0 keys | 5462 slots | 1 slaves.
192.168.27.13:6379 (87d5d927...) -> 0 keys | 5461 slots | 1 slaves.
[OK] 0 keys in 3 masters.
0.00 keys per slot on average.
```

```bash
[root@slave1 ~]# redis-cli 
127.0.0.1:6379> auth 111111
OK
127.0.0.1:6379> info replication
# Replication
role:slave
master_host:192.168.27.13
master_port:6379
master_link_status:up
master_last_io_seconds_ago:4
master_sync_in_progress:0
slave_repl_offset:2380
slave_priority:100
slave_read_only:1
connected_slaves:0
master_replid:0d65ecf93add37a70309658508ec3d3ca4cc3186
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:2380
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:2380
```
```bash
127.0.0.1:6379> CLUSTER info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:3
cluster_stats_messages_ping_sent:1768
cluster_stats_messages_pong_sent:1726
cluster_stats_messages_meet_sent:1
cluster_stats_messages_sent:3495
cluster_stats_messages_ping_received:1722
cluster_stats_messages_pong_received:1769
cluster_stats_messages_meet_received:4
cluster_stats_messages_received:3495

```

```bash
127.0.0.1:6379> CLUSTER NODES
7641530f649de98c30f4f02cdc42c299f4c339e4 192.168.27.15:6379@16379 slave 58e21837e91adc28f55cd909e22d610ba4d5e18c 0 1560479614000 5 connected
3376d3a9d73316075955fb59afa6e03b525419a9 192.168.27.12:6379@16379 master - 0 1560479614430 2 connected 5461-10922
58e21837e91adc28f55cd909e22d610ba4d5e18c 192.168.27.11:6379@16379 master - 0 1560479614000 1 connected 0-5460
cc3d1d0c9c64bbce0587507f5d4b0e3233691e2f 192.168.27.16:6379@16379 slave 3376d3a9d73316075955fb59afa6e03b525419a9 0 1560479613424 2 connected
87d5d9278ed230d0817ac36e59dd2d39487e878c 192.168.27.13:6379@16379 master - 0 1560479615436 3 connected 10923-16383
b6406a84234193e958011b21b01c410aed019091 192.168.27.14:6379@16379 myself,slave 87d5d9278ed230d0817ac36e59dd2d39487e878c 0 1560479612000 4 connected

```
从服务器无法写入数据，只能在主服务器写入数据
```bash
(error) MOVED 13252 192.168.27.13:6379
127.0.0.1:6379> set key3 v1
```

测试让master停止服务看slave能否顶上
在redis1上停止服务
```bash
[root@redis1 ~]# ps aux | grep redis
root      21450  0.2  0.4 153504  2100 ?        Ssl  09:23   0:11 redis-server 0.0.0.0:6379 [cluster]
root      36460  0.0  0.2 112708   976 pts/0    R+   10:39   0:00 grep --color=auto redis
[root@redis1 ~]# kill -9 21450
```
在slave2上查看日志
```bash
[root@slave2 ~]# cat /apps/redis/logs/redis.log 
8406:S 14 Jun 10:03:08.339 * Full resync from master: a16884ef7c43900ef612b43c0474bbd4e555ca2e:0
8406:S 14 Jun 10:03:08.339 * Discarding previously cached master state.
8406:S 14 Jun 10:03:08.373 * MASTER <-> SLAVE sync: receiving 176 bytes from master
8406:S 14 Jun 10:03:08.373 * MASTER <-> SLAVE sync: Flushing old data
8406:S 14 Jun 10:03:08.373 * MASTER <-> SLAVE sync: Loading DB in memory
8406:S 14 Jun 10:03:08.374 * MASTER <-> SLAVE sync: Finished with success
8406:S 14 Jun 10:39:27.286 # Connection with master lost.                               #无法连接到主服务器
8406:S 14 Jun 10:39:27.286 * Caching the disconnected master state.
8406:S 14 Jun 10:39:28.190 * Connecting to MASTER 192.168.27.11:6379
8406:S 14 Jun 10:39:28.190 * MASTER <-> SLAVE sync started
8406:S 14 Jun 10:39:28.191 # Error condition on socket for SYNC: Connection refused     #同步失败
                ....中间省略....                                                         
8406:S 14 Jun 10:39:43.349 * Connecting to MASTER 192.168.27.11:6379
8406:S 14 Jun 10:39:43.349 * MASTER <-> SLAVE sync started
8406:S 14 Jun 10:39:43.350 # Error condition on socket for SYNC: Connection refused
8406:S 14 Jun 10:39:44.171 * FAIL message received from 87d5d9278ed230d0817ac36e59dd2d39487e878c about 58e21837e91adc28f55cd909e22d610ba4d5e18c
8406:S 14 Jun 10:39:44.171 # Cluster state changed: fail                                    
8406:S 14 Jun 10:39:44.260 # Start of election delayed for 893 milliseconds (rank #0, offset 3024).
8406:S 14 Jun 10:39:44.362 * Connecting to MASTER 192.168.27.11:6379
8406:S 14 Jun 10:39:44.362 * MASTER <-> SLAVE sync started
8406:S 14 Jun 10:39:44.362 # Error condition on socket for SYNC: Connection refused
8406:S 14 Jun 10:39:45.168 # Starting a failover election for epoch 7.                  #经过一段时间失败后，开始故障转移自己变为主服务器
8406:S 14 Jun 10:39:45.174 # Failover election won: I'm the new master.
8406:S 14 Jun 10:39:45.175 # configEpoch set to 7 after successful failover
8406:M 14 Jun 10:39:45.175 # Setting secondary replication ID to a16884ef7c43900ef612b43c0474bbd4e555ca2e, valid up to offset: 3025. New replication ID is 86b442d576231b75ae48248b82436fa034de439e
8406:M 14 Jun 10:39:45.175 * Discarding previously cached master state.
8406:M 14 Jun 10:39:45.175 # Cluster state changed: ok

```
查看slave2的状态
```bash
[root@slave2 ~]# redis-cli 
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:0
master_replid:86b442d576231b75ae48248b82436fa034de439e
master_replid2:a16884ef7c43900ef612b43c0474bbd4e555ca2e
master_repl_offset:3024
second_repl_offset:3025
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:3024
```
将redis1再次启动，登录后查看服务器状态
```bash
root@redis1 ~]# redis-server /apps/redis/etc/redis.conf     
[root@redis1 ~]# redis-cli          #登录redis
127.0.0.1:6379> auth 111111
OK
127.0.0.1:6379> info replication
# Replication
role:slave                          #已百变为slave
master_host:192.168.27.15           #主服务器为slave2
master_port:6379
master_link_status:up
master_last_io_seconds_ago:10
master_sync_in_progress:0
slave_repl_offset:3066
slave_priority:100
slave_read_only:1
connected_slaves:0
master_replid:86b442d576231b75ae48248b82436fa034de439e
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:3066
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:3025
repl_backlog_histlen:42
```