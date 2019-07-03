# redis 集群

- 实验高可用性和高并发
  1. master 和 slave 角色的无缝切换，让业务无感知从而不影响业务使用
  2. 可以横向动态扩展 Redis 服务器，从而实现多台服务器并行写入以实现更高并发的目的。

- Redis 集群实现方式：客户端分片 代理分片 Redis Cluster

## 1.Sentinel(哨兵）

- Sentinel 进程是用于监控 redis 集群中 Master 主服务器工作的状态，在 Master 主服务器发生故障的时候，可以实现 Master 和 Slave 服务器的切换，保证系统的高可用，其已经被集成在 redis2.6+的版本中，Redis 的哨兵模式到了 2.8 版本之后就稳定了下来。一般在生产环境也建议使用 Redis 的 2.8 版本的以后版本。
- 哨兵(Sentinel) 是一个分布式系统，可以在一个架构中运行多个哨兵(sentinel) 进程，这
  些进程使用流言协议(gossip protocols)来接收关于 Master 主服务器是否下线的信息，并使用投票协议(Agreement Protocols)来决定是否执行自动故障迁移,以及选择哪个 Slave 作为新的 Master。每个哨兵(Sentinel)进程会向其它哨兵(Sentinel)、Master、Slave 定时发送消息，以确认对方是否”活”着，如果发现对方在指定配置时间(可配置的)内未得到回应，则暂时认为对方已掉线，也就是所谓的”主观认为宕机” ，主观是每个成员都具有的独自的而且可能相同也可能不同的意识，英文名称：SubjectiveDown，简称 SDOWN。有主观宕机，肯定就有客观宕机。当“哨兵群”中的多数 Sentinel 进程在对 Master主服务器做出 SDOWN 的判断，并且通过 SENTINEL is-master-down-by-addr 命令互相交流之后，得出的 Master Server 下线判断，这种方式就是“客观宕机”，客观是不依赖于某种意识而已经实际存在的一切事物，英文名称是：Objectively Down， 简称 ODOWN。
- 通过一定的 vote 算法，从剩下的 slave 从服务器节点中，选一台提升为 Master 服务器节点，然后自动修改相关配置，并开启故障转移（failover）。
- Sentinel 机制可以解决 master 和 slave 角色的切换问题。

### 1.1哨兵的架构

- 需要手动先指定某一台 Redis 服务器为 master，然后将其他 slave 服务器使用命令配置为 master 服务器的 slave,哨兵的前提是已经手动实现了一个 redis master-slave 的运行环境。
- 哨兵只能解决高可用性，不能实现高并发。

```bash
#部署服务器 主从
Redis Master 192.168.1.18  主
Redis Slave  192.168.1.28  从
Redis Slave  192.168.1.38  从
```

![1560488191366](C:\Users\L\AppData\Roaming\Typora\typora-user-images\1560488191366.png)

### 1.2 搭建master

```bash
# 安装最小系统的软件包
yum install  vim iotop bc gcc gcc-c++ glibc glibc-devel pcre pcre-devel openssl  openssl-devel zip unzip zlib-devel  net-tools lrzsz tree ntpdate telnet lsof tcpdump wget libevent libevent-devel bc  systemd-devel bash-completion traceroute -y
 #获取官网的安装包
  wget http://download.redis.io/releases/redis-4.0.14.tar.gz
 #进入目录解压安装包
  cd /usr/local/src
  tar xf redis-4.0.14.tar.gz 
 #进入解压文件
  cd redis-4.0.14/
 #编译安装，指定安装的位置
 make PREFIX=/apps/redis install
 #生成配置文件目录，并且从解压文件里面拷贝配置文件到创建的目录
  mkdir /apps/redis/etc/
  cp redis.conf /apps/redis/etc/
 
 #启动
/apps/redis/bin/redis-server /apps/redis/etc/redis.conf
 #关闭服务（通过杀进程的方式）
kill -9  `cat /apps/redis/run/redis_6379.pid`
 
 #解决启动警告问题
vim /etc/sysctl.conf
net.core.somaxconn = 512
vm.overcommit_memory = 1
#让前面两个修改生效
sysctl -p

#永久修改
vim /etc/rc.d/rc.local 
echo never >/sys/kernel/mm/transparent_hugepage/enabled
#开机加载文件，没有执行权限
chmod a+x /etc/rc.d/rc.local 
#重启,生效
reboot

#编辑启动脚本
[root@CentOS7 ~]#vim /usr/lib/systemd/system/redis.service
[Unit]
Description=Redis persistent key-value database
After=network.target
After=network-online.target
Wants=network-online.target
[Service]
#ExecStart=/usr/bin/redis-server /etc/redis.conf --supervised systemd
ExecStart=/apps/redis/bin/redis-server /apps/redis/etc/redis.conf --supervised systemd
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
Type=notify
User=redis    # 将root修改为redis，root权限太大，会给黑客漏洞。
Group=redis   #所属组也要修改
RuntimeDirectory=redis
RuntimeDirectoryMode=0755
[Install]
WantedBy=multi-user.target

#添加用户，修改所有者，所属组
useradd redis -s /sbin/nologin
chown redis.redis /apps/redis/ -R
#启动服务
systemctl start redis.service 
ss -ntl
ps -ef |grep redis

#创立软连接
ln -sv /apps/redis/bin/redis-* /usr/bin/

#配置文件
[root@CentOS7 ~]#grep "^[a-Z]" /apps/redis/etc/redis.conf 
主master不需要修改（自己查看，IP地址等还是需要添加修改的）

```

### 1.3服务器配置 slave:

- 192.168.1.28 ,192.168.1.38

```bash
#将master服务打包拷贝过来 192.168.1.18
tar -cvf /apps/redis.tar /apps/redis
scp /apps/redis.tar 192.168.1.28:/apps/

#slave server：192.168.1.28
useradd reids /sbin/nologin
tar xf /apps/redis.tar
# 复制启动文件
scp /usr/lib/systemd/system/redis.service
192.168.1.28:/usr/lib/systemd/system/
ln -sv /apps/redis/bin/redis-* /usr/bin/



#修改从服务器配置文件
vim /apps/redis/etc/redis.conf

slaveof 192.168.1.18 6379

masteraut 123456

启动
/apps/redis/bin/redis-server /apps/redis/etc/redis.conf
 #关闭服务（通过杀进程的方式）
kill -9  `cat /apps/redis/run/redis_6379.pid`
```

### 1.4 配置哨兵

- 哨兵可以不和 Redis 服务器部署在一起，也可以放一起。
- 监控地址可以是多个（一般是一个），每个服务的端口必须要一样。

```bash
#server1 的配置
[root@redis-s1 etc]# grep "^[a-Z]" /apps/redis/etc/sentinel.conf 
bind 192.168.1.18  
port 26379
daemonize yes
pidfile "redis-sentinel.pid"
logfile "sentinel_26379.log"
dir "/apps/redis/logs/redis"
sentinel monitor mymaster 192.168.1.18 6379 2 #法定人数限制(quorum)，即有几个 slave 认为 masterdown 了就进行故障转移，选举
sentinel auth-pass mymaster 123456
sentinel down-after-milliseconds mymaster 10000 #(SDOWN)主观下线的时间，10秒没反应就认为down了
sentinel parallel-syncs mymaster 1 #发生故障转移时候同时向新 master 同步数据的 slave 数量，数字越小总同步时间越长
sentinel failover-timeout mymaster 180000 #所有 slaves 指向新的 master 所需的超时时间
sentinel deny-scripts-reconfig yes #禁止修改脚本

#server2 的配置
bind 192.168.1.28                     
port 26379
daemonize yes 
pidfile "redis-sentinel.pid"
logfile "sentinel_26379.log"
dir "/apps/redis/logs"
sentinel myid 1b8c875c6bf4e50fe0944c604c601ae396d30ec9
sentinel deny-scripts-reconfig yes 
sentinel monitor mymaster 192.168.1.18 6379 2
sentinel down-after-milliseconds mymaster 10000
sentinel auth-pass mymaster 123456
sentinel config-epoch mymaster 0


#Server3 配置：
bind 192.168.1.38                                                                                  
port 26379
daemonize yes 
pidfile "redis-sentinel.pid"
logfile "sentinel_26379.log"
dir "/apps/redis/logs"
sentinel myid d96f7ccb38a664e2289489cb0cd764d6f0944564
sentinel deny-scripts-reconfig yes 
sentinel monitor mymaster 192.168.1.18 6379 2
sentinel down-after-milliseconds mymaster 10000
sentinel auth-pass mymaster 123456
sentinel config-epoch mymaster 0


```

### 1.4 启动哨兵

```bash
三台哨兵都要启动
#/apps/redis/bin/redis-sentinel /apps/redis/etc/sentinel.conf
#/apps/redis/bin/redis-sentinel /apps/redis/etc/sentinel.conf
#/apps/redis/bin/redis-sentinel /apps/redis/etc/sentinel.conf
```



### 1.5 验证和查看

```bash
#验证端口
ss -ntl
LISTEN      0      511        192.168.1.18:26379                           

#查看哨兵日志
tail -f /apps/redis/logs/sentinel_26379.log 

#查看你当前redis状态
redis-cli
info Replication
# Replication
role:master
connected_slaves:2
slave0:ip=192.168.1.28,port=6379,state=online,offset=117011,lag=0
slave1:ip=192.168.1.38,port=6379,state=online,offset=117150,lag=0
master_replid:c878ab84d5de34b425bc418b60cddedafc4b9069
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:117150
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:117150

#当前sentinel状态 哨兵
redis-cli  -h 192.168.1.18 -p 26379
	# Sentinel
	sentinel_masters:1
	sentinel_tilt:0
	sentinel_running_scripts:0
	sentinel_scripts_queue_length:0
	sentinel_simulate_failure_flags:0
	master0:name=mymaster,status=ok,address=192.168.1.18:6379,slaves=2,sentinels=3


```





### 1.6测试哨兵

- 停止Reids Master 测试故障转移

```bash
[root@redis-s1 ~]# systemctl stop redis
#查看集群信息：
[root@redis-s1 ~]# redis-cli -h 192.168.1.28 -p 6379
# Replication
role:master
connected_slaves:1
slave0:ip=192.168.1.38,port=6379,state=online,offset=149367,lag=1
master_replid:0a88d94d5c624f5adba8f131b3b316525480adc9
master_replid2:c878ab84d5de34b425bc418b60cddedafc4b9069
master_repl_offset:149367
second_repl_offset:133665
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:149367



#查看哨兵信息
[root@redis-s1 ~]# redis-cli -h 192.168.7.101 -p 26379
# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=mymaster,status=ok,address=192.168.7.101:6379,slaves=2,sentinels=3

#故障转移后的 redis 配置文件： 
故障转移后 redis.conf 中的 replicaof 行的 master IP 会被修改，sentinel.conf 中的 sentinel monitor IP 会被修改。
#查看
[root@CentOS7 redis]#grep "^[a-Z]" /apps/redis/etc/sentinel.conf 
bind 192.168.1.28
port 26379
daemonize yes
pidfile "redis-sentinel.pid"
logfile "sentinel_26379.log"
dir "/apps/redis/logs"
sentinel myid 1b8c875c6bf4e50fe0944c604c601ae396d30ec9  #修改当前
sentinel deny-scripts-reconfig yes
sentinel monitor mymaster 192.168.1.28 6379 2
sentinel down-after-milliseconds mymaster 10000
sentinel auth-pass mymaster 123456
sentinel config-epoch mymaster 1
sentinel leader-epoch mymaster 1  后面几行都被修改
sentinel known-slave mymaster 192.168.1.38 6379
sentinel known-slave mymaster 192.168.1.18 6379
sentinel known-sentinel mymaster 192.168.1.38 26379 d96f7ccb38a664e2289489cb0cd764d6f0944564
sentinel known-sentinel mymaster 192.168.1.18 26379 a831e227a8d567b49b4b23de8ae7c56194c79b9d
sentinel current-epoch 1


#在此恢复机器
systemctl restart redis

#加入需要在此打开主从
vim /apps/redis/etc/redis.conf 
slaveof 192.168.1.28 
masterauth 123456
 #重启服务
 systemctl restart redis
 
 
 #服务器状态
 192.168.1.18   从
 192.168.1.28   主
 192.168.1.38    从
 
```

- （咧逗是linux里redis哨兵，不是斯坦·李老爷子里面的那个哨兵，也不是LOL里面曾经的哨兵之殇）。

## 2.Redis Cluster：集群

### 2.1简介

- 集群能实现，高可用性和高并发。
- Redis 分布式部署方案：
  1. 客户端分区：由客户端程序决定 key 写分配和写入的 redis node，但是需要客户端自己处理写入分配、高可用管理和故障转移等
  2. 代理方案：基于三方软件实现 redis proxy，客户端先连接之代理层，由代理层实现 key 的写入分配，对客户端来说是有比较简单，但是对于集群管节点增减相对比较麻烦，而且代理本身也是单点和性能瓶颈。

- 在哨兵 sentinel 机制中，可以解决 redis 高可用的问题，即当 master 故障后可以自动将 slave 提升为master 从而可以保证 redis 服务的正常使用，但是无法解决 redis 单机写入的瓶颈问题，**即单机的 redis写入性能受限于单机的内存大小、并发数量、网卡速率等因素**，因此 redis 官方在 redis 3.0 版本之后推出了无中心架构的 redis cluster 机制，在无中心的 redis 集群汇中，**其每个节点保存当前节点数据和整个集群状态**,每个节点都和其他所有节点连接，特点如下

  1. 所有 Redis 节点使用(PING 机制)互联
  2. 集群中某个节点的失效，是整个集群中超过半数的节点监测都失效才算真正的失效
  3. 客户端不需要 proxy 即可直接连接 redis，应用程序需要写全部的 redis 服务器 IP。
  4. redis cluster 把所有的 redis node 映射到 0-16383 个槽位(slot)上，读写需要到指定的 redis node 上进行操作，因此有多少个 reids node 相当于 redis 并发扩展了多少倍
  5. Redis cluster 预先分配 16384 个(slot)槽位，当需要在 redis 集群中写入一个 key -value 的时候，会使用 CRC16(key) mod 16384 之后的值，决定将 key 写入值哪一个槽位从而决定写入哪一个 Redis 节点上，从而有效解决单机瓶颈。

### 2.2 Redis cluster架构





### 2.3 部署 redis 集群

- 环境准备

```bash
#环境：7台服务器
192.168.1.18：6379	主
192.168.1.28：6379	主
192.168.1.38：6379	主
192.168.1.48：6379	从
192.168.1.58：6379	从
192.168.1.68：6379	从
#备用服务器一台：
集群添加节点测试
192.168.1.78：6379	主
192.168.1.78：6380	从
```

### 2.4 创建 redis cluster 集群的前提

1. 每个 redis node 节点采用相同的硬件配置、相同的密码
2. 每个节点必须开启的参数
     cluster-enabled yes #必须开启集群状态，开启后 redis 进程会有 cluster 显示
     cluster-config-file nodes-6380.conf #此文件有 redis cluster 集群自动创建和维护，不需要任何手动操作
3. 所有 redis 服务器必须没有任何数据
4. 先启动为单机 redis 且没有任何 key value
5. 三台服务器主备存放是交叉的，不然一个 服务器down，整个服务器就downl。

### 2.5 创建集群：

```bash
#搭建redis
#编译安装
# 安装最小系统的软件包
yum install  vim iotop bc gcc gcc-c++ glibc glibc-devel pcre pcre-devel openssl  openssl-devel zip unzip zlib-devel  net-tools lrzsz tree ntpdate telnet lsof tcpdump wget libevent libevent-devel bc  systemd-devel bash-completion traceroute -y
 #获取官网的安装包
  wget http://download.redis.io/releases/redis-4.0.14.tar.gz
 #进入目录解压安装包
  cd /usr/local/src
  tar xf redis-4.0.14.tar.gz 
 #进入解压文件
  cd redis-4.0.14/
 #编译安装，指定安装的位置
 make PREFIX=/apps/redis install
 #生成配置文件目录，并且从解压文件里面拷贝配置文件到创建的目录
  mkdir /apps/redis/etc/
  cp redis.conf /apps/redis/etc/
 
 #启动
/apps/redis/bin/redis-server /apps/redis/etc/redis.conf
 #关闭服务（通过杀进程的方式）
kill -9  `cat /apps/redis/run/redis_6379.pid`
 
 #解决启动警告问题
vim /etc/sysctl.conf
net.core.somaxconn = 512
vm.overcommit_memory = 1
#让前面两个修改生效
sysctl -p

#永久修改
vim /etc/rc.d/rc.local 
echo never >/sys/kernel/mm/transparent_hugepage/enabled
#开机加载文件，没有执行权限
chmod a+x /etc/rc.d/rc.local 
#重启,生效
reboot

#编辑启动脚本
[root@CentOS7 ~]#vim /usr/lib/systemd/system/redis.service
[Unit]
Description=Redis persistent key-value database
After=network.target
After=network-online.target
Wants=network-online.target
[Service]
#ExecStart=/usr/bin/redis-server /etc/redis.conf --supervised systemd
ExecStart=/apps/redis/bin/redis-server /apps/redis/etc/redis.conf --supervised systemd
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
Type=notify
User=redis  
Group=redis 
RuntimeDirectory=redis
RuntimeDirectoryMode=0755
[Install]
WantedBy=multi-user.target

#添加用户，修改所有者，所属组
useradd redis -s /sbin/nologin
chown redis.redis /apps/redis/ -R
#启动服务
systemctl start redis.service 
ss -ntl
ps -ef |grep redis

#创立软连接
ln -sv /apps/redis/bin/redis-* /usr/bin/

```

### 2.3

```bash
#修改配置文件
vim /apps/redis/etc/redis.conf
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
#每个节点都要开启
cluster-enabled yes #必须开启集群状态，开启后redis进程会有cluster显示 
cluster-config-file nodes-6380.conf #此文件有redis cluster集群自动创建和维护，不需要任何手动操作

#创建文件传递到每个节点上
scp /apps/redis 192.168.1.18:/apps

#在所有redis服务器上启动服务
redis-server /apps/redis/etc/redis.conf 

#查看所有主机上集群服务器的16379端口是否开启

```



### 2.3 redis -trib.rb 命令

```bash
#解压源码包
tar -xf ruby-2.5.5.tar.gz 

#检查编译环境和编译所需要的组件
./configure 
#编译安装
make && make install

#ruby编译完成后还需要安装一个redis的模块
gem install redis

#建立软连接
ln -sv /usr/local/src/redis-4.0.14/src/redis-trib.rb /usr/bin

#验证redis-trib.rb 是否可以执行
 [root@CentOS7 src]# redis-trib.rb 
  Usage: redis-trib <command> <options> <arguments ...>
 create host1:port1 ... hostN:portN #创建集群
 --replicas <arg> #指定 master 的副本数量
 check host:port #检查集群信息
 info host:port #查看集群主机信息
 fix host:port #修复集群
 --timeout <arg>
 reshard host:port #在线热迁移集群指定主机的 slots 数据
 --from <arg>
 --to <arg>
 --slots <arg>
 --yes
 --timeout <arg>
 --pipeline <arg>
 rebalance host:port #平衡集群中各主机的 slot 数量
 --weight <arg>
 --auto-weights
 --use-empty-masters
 --timeout <arg>
 --simulate
 --pipeline <arg>
 --threshold <arg>
 add-node new_host:new_port existing_host:existing_port #添加主机到集群
 --slave
 --master-id <arg>
 del-node host:port node_id #删除主机
 set-timeout host:port milliseconds #设置节点的超时时间
 call host:port command arg arg .. arg #在集群上的所有节点上执行命令
 import host:port #导入外部 redis 服务器的数据到当前集群
 --from <arg>
 --copy
 --replace
 help (show this help)
 
 
```

1.  下面创建集群会遇到的第一个坑，修改密码为redis 登录密码，改成节点连接密码
   vim /usr/local/lib/ruby/gems/2.5.0/gems/redis-4.1.2/lib/redis/client.rb 

![1560516122230](C:\Users\L\AppData\Roaming\Typora\typora-user-images\1560516122230.png)

2. 没有打开集群

![1560518772761](C:\Users\L\AppData\Roaming\Typora\typora-user-images\1560518772761.png)

3. 没有打开集群
   cluster-enabled yes 
   cluster-config-file nodes-6380.conf

4. 如果有之前的操作导致 Redis 集群创建报错，
   先清空文件

   > /apps/redis/logs/nodes-6379.conf 
   > #则执行清空数据和集群命令：
   > 127.0.0.1:6379> FLUSHALL
   > OK
   > 127.0.0.1:6379> cluster reset
   > OK

### 2.3 创建 redis cluster 集群：

```bash
redis-trib.rb create --replicas 1 192.168.1.18:6379 192.168.1.28:6379 192.168.1.38:6379 192.168.1.7:6379 192.168.1.58:6379 192.168.1.68:6379
 yes


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

- 简洁的显示集群信息

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



从服务器无法写入数据，只能在主服务器写入数据

```bash
(error) MOVED 13252 192.168.27.13:6379
127.0.0.1:6379> set key3 v1
```

测试让master停止服务看slave能否顶上,在redis1上停止服务

```bash
[root@redis1 ~]# ps aux | grep redis
root      21450  0.2  0.4 153504  2100 ?        Ssl  09:23   0:11 redis-server 0.0.0.0:6379 [cluster]
root      36460  0.0  0.2 112708   976 pts/0    R+   10:39   0:00 grep --color=auto redis
[root@redis1 ~]# kill -9 21450
```

```bash
#在slave2上查看日志

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
```



- 将redis1再次启动，登录后查看服务器状态

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











### 2.3

























### 2.3