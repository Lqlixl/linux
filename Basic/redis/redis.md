# Redis

##  1.1 redis 基础：

- 官网地址：https://redis.io/

- Redis和Memcached是非关系型数据库也称为NoSQL数据库，MySQL、Mariadb、SQL Server、PostgreSQL、Oracle 数据库属于关系型数据(RDBMS, Relational Database Management System)

- 缓存 

  - cache读缓存
  - buffers 写缓存，缓冲区

  

### 1.1.1 redis简介

- Redis(Remote Dictionary Server)在 2009 年发布，开发者 Salvatore Sanfilippo 是意大利开发者，他本想为自己的公司开发一个用于替换 MySQL 的产品 Redis，但是没有想到他把 Redis 开源后大受欢迎，短
  短几年，Redis 就有了很大的用户群体，目前国内外使用的公司有知乎网、新浪微博、GitHub 等redis是一个开源的、遵循BSD协议的、基于内存的而且目前比较流行的键值数据库(key-value database)，是一个非关系型数据库，redis 提供将内存通过网络远程共享的一种服务，提供类似功能的还有memcache，但相比 memcache，redis 还提供了易扩展、高性能、具备数据持久性等功能。
- Redis 在高并发、低延迟环境要求比较高的环境使用量非常广泛，目前 redis 在 DB-Engine 月排行榜https://db-engines.com/en/ranking 中一直比较靠前，而且一直是键值型存储类的首位。

### 1.1.2 redis和memcached的对比：

- 支持数据的持久化：可以将内存中的数据保持在磁盘中，重启 redis 服务或者服务器之后可以从备份文件中恢复数据到内存继续使用。
- 支持更多的数据类型：支持 string(字符串)、hash(哈希数据)、list(列表)、set(集合)、zet(有序集合)
- 支持数据的备份：可以实现类似于数据的 master-slave 模式的数据备份，另外也支持使用快照+AOF。
- 支持更大的 value 数据：memcache 单个 key value 最大只支持 1MB，而 redis 最大支持 512MB。
- Redis 是单线程，而 memcache 是多线程 所以单机情况下没有 memcache 并发高，但 redis 支持分布式集群以实现更高的并发，单 Redis 实例可以实现数万并发。
- 支持集群横向扩展：基于 redis cluster 的横向扩展，可以实现分布式集群，大幅提升性能和数据安全性。
- 都是基于 C 语言开发。

### 1.1.3 redis 典型应用场景

- Session 共享：常见于 web 集群中的 Tomcat 或者 PHP 中多 web 服务器 session 共享
- 消息队列：ELK 的日志缓存、部分业务的订阅发布系统
- 计数器：访问排行榜、商品浏览数等和次数相关的数值统计场景
- 缓存：数据查询、电商网站商品信息、新闻内容
- 微博/微信社交场合：共同好友、点赞评论等

## 2.1 安装及使用

- 官方下载地址：
  - http://download.redis.io/releases/

### 2.2.1： yum 安装redis

```bash
#在centos系统上需要安装 epel 源。
yum install list redis  #查看版本
yum install redis -y #安装
redis-cli     #进入
```



### 2.2.2编译安装redis

- 官网下载当前最新 release 版本 redis 源码包：
  - http://download.redis.io/releases/

### 2.2.3编译安装及命令

- 官方的安装命令：
  - https://redis.io/download

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
  #查看配置文件
  grep "^[a-Z]" /apps/redis/etc/redis.conf 
  ...
  ...
 
```

### 2.2.4在前台启动redis

```bash
#启动
/usr/local/redis/bin/redis-server /usr/local/redis/etc/redis.conf
#安装的时候换了目录，所以前台启动目录修改
/apps/redis/bin/redis-server /apps/redis/etc/redis.conf
```

### 2.2.5 解决当前警告的提示

![1560306933660](D:\学习资料\markdow\1560306933660.png)

```bash
 vim /etc/sysctl.conf    #在最后增加以下内容
#解决第一个报警warning
  #最大队列长度，应付突发的大并发连接请求
  net.core.somaxconn = 512
  #半连接队列长度，此值受限于内存大小
  net.ipv4.tcp_max_syn_backlog = 550

#解决第二个报警warning
 #内存分配策略
 vm.overcommit_memory = 1
 
 说明：overcommit_memory的值有三个，0、1、2。 默认为0
0， 表示内核将检查是否有足够的可用内存供应用进程使用；如果有足够的可用内存，内存申请允许；否则，内存申请失败，并把错误返回给应用进程。 
1， 表示内核允许分配所有的物理内存，而不管当前的内存状态如何，这里表示的所有内存是受限于overcommit_ratio值的，值是一个百分比，在Debian8上/proc/sys/vm/overcommit_ratio的默认值是50，那把overcommit_memory配置为1时，那系统可分配的内存为“物理内存+物理内存*50%”
2， 表示内核允许分配超过所有物理内存和交换空间总和的内存

#查看添加的配置参数
cat /etc/sysctl.conf
net.core.somaxconn = 512
vm.overcommit_memory = 1

#让前面两个修改生效
sysctl -p


# 解决第三个报警warning
 #关于大页问题，
 内存可管理的最小单位是page，一个page通常是4kb，那1M内存就会有256个page，CPU通过内置的内存管理单元管理page表记录。Huge Pages就是表示page的大小已超过4kb了，一般是2M到1G，它的出现主要是为了管理超大内存。THP会导致内存锁影响性能，所以一般建议关闭此功能。
#“/sys/kernel/mm/transparent_hugepage/enabled”有三个值：如下
cat /sys/kernel/mm/transparent_hugepage/enabled
 always [madvise] never
 # always 尽量使用透明内存，扫描内存，有512个 4k页面可以整合，就整合成一个2M的页面
 # never 关闭，不使用透明内存
 # madvise 避免改变内存占用
 
 #临时修改
 echo never >/sys/kernel/mm/transparent_hugepage/enabled
 
#永久修改
vim /etc/rc.d/rc.local 
echo never >/sys/kernel/mm/transparent_hugepage/enabled
#开机加载文件，没有执行权限
chmod a+x /etc/rc.d/rc.local 

#重启,生效
reboot
 
```

### 2.2.6：编辑redis服务启动脚本

```bash
#关闭服务（通过杀进程的方式）
kill -9  `cat /apps/redis/run/redis_6379.pid`
#打开服务
/apps/redis/bin/redis-server /apps/redis/etc/redis.conf


#添加启动脚本
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
```



### 2.3使用redis

#### 2.3.1创建命令软连接： 
```bash
#创立软连接
ln -sv /apps/redis/bin/redis-* /usr/bin/
#编译安装后的命令
[root@CentOS7 ~]#ll /apps/redis/bin/
redis-benchmark #reids性能检测工具
redis-check-aof #AOF 文件检查工具
redis-check-rdb  #RDB  文件检查工具
redis-cli   #redis 客服端工具
redis-sentinel -> redis-server  #哨兵，软连接到server  
redis-server  #redis 服务端

#修复文件aof 和rdb
redis-check-aof --fix /apps/redis/data/appendoly_6379.aof
redis-check-rdb /apps/redis/data/dump_6379.rdb
```

#### 2.3.2使用客服端来连接redis：

```bash
#运维连接和登录

#redis查询使用帮助
	redis-cli -h
#修改配置文件，添加IP让其他客服端连接
vim /apps/redis/etc/redis.conf
	bind 127.0.0.1 192.168.1.18  #监听地址，可以用空格隔开后多个监听 IP
#重启服务
systemctl restart redis.service 

#本机非密码连接
  redis-cli
#跨主机登录
  redis-cli -h HOSTNAME/IP -p PORT -a PASSWORD
   redis-cli -h 192.168.1.18
   
#开发连接和登录
#!/bin/env python
#Author: lqlixl
import redis
import time
pool = redis.ConnectionPool(host="192.168.7.101",
port=6379,password="")
r = redis.Redis(connection_pool=pool)
for i in range(100):
 r.set("k%d" % i,"v%d" % i)
 time.sleep(1)
 data=r.get("k%d" % i)
 print(data)

```

### 3.Redis配置文件

```bash
[root@CentOS7 ~]#grep "^[a-Z]" /apps/redis/etc/redis.conf 

bind 127.0.0.1 172.20.76.18  #监听地址，可以用空格隔开后多个监听 IP
protected-mode yes #redis3.2 之后加入的新特性，在没有设置 bind IP 和密码的时候只允许访问127.0.0.1:6379
port 6379    #监听端口
tcp-backlog 511   #三次握手的时候 server 端收到 client ack 确认号之后的队列值。
timeout 0    #客户端和 Redis 服务端的连接超时时间，默认是 0，表示永不超时。
tcp-keepalive 300 #tcp 会话保持时间
daemonize no #认情况下 redis 不是作为守护进程运行的，如果你想让它在后台运行，你就把它改成yes,当 redis 作为守护进程运行的时候，它会写一个 pid 到 /var/run/redis.pid 文件里面
supervised no #和操作系统相关参数，可以设置通过 upstart 和 systemd 管理 Redis 守护进程，centos 7以后都使用 systemd
pidfile /apps/redis/run/redis_6379.pid  #pid 文件路径
loglevel notice #日志级别
logfile “/apps/redis/logs/redis_6379.log” #日志路径

# 需要建立目录
mkdir /apps/redis/{data,run,logs}
[root@CentOS7 ~]#ll /apps/redis/
total 0
drwxr-xr-x 2 redis redis 134 Jun 12 09:35 bin  #开机目录
drwxr-xr-x 2 root  root    6 Jun 12 21:55 data #存放数据的目录
drwxr-xr-x 2 redis redis  47 Jun 12 21:49 etc #存放配置文件的目录
drwxr-xr-x 2 root  root    6 Jun 12 21:55 run  #运行目录
drwxr-xr-x 2 root  root    6 Jun 12 22:15 logs #日志目录
#重启
redis-server /apps/redis/etc/redis.conf
ps -ef |grep redis
ss -ntl


databases 16 #设置 db 库数量，默认 16 个库，默认为（0~15），切换数据库，都可以有与其他库相同的键值对，每个库的键值对是唯一的，可以修改。
always-show-logo yes #在启动 redis 时是否显示 log

#快照
save <seconds> <changes>
save 900 1 #在 900 秒内有一个键内容发生更改就出就快照机制
save 300 10  #300秒内有10个更改,将内存中的数据快照写入磁盘。
save 60 10000  #60秒内有10000个更改,将内存中的数据快照写入磁盘。
stop-writes-on-bgsave-error no #快照出错时是否禁止 redis 写入操作，
rdbcompression yes #持久化到 RDB 文件时，是否压缩，"yes"为压缩，"no"则反之
rdbchecksum yes #是否开启 RC64 校验，默认是开启
dbfilename dump_6379.rdb #快照文件名
dir /apps/redis/data #快照文件保存路径

#slave从库
replica-serve-stale-data yes #当从库同主库失去连接或者复制正在进行，从机库有两种运行方式：1) 如果 replica-serve-stale-data 设置为 yes(默认设置)，从库会继续响应客户端的读请求。2) 如果 replicaserve-stale-data 设置为 no，除去指定的命令之外的任何请求都会返回一个错误"SYNC with master inprogress"。
replica-read-only yes #是否设置从库只读
repl-diskless-sync no #是否使用 socket 方式复制数据，目前 redis 复制提供两种方式，disk 和 socket，如果新的 slave 连上来或者重连的 slave 无法部分同步，就会执行全量同步，master 会生成 rdb 文件，有2 种方式：disk 方式是 master 创建一个新的进程把 rdb 文件保存到磁盘，再把磁盘上的 rdb 文件传递给 slave，socket 是 master 创建一个新的进程，直接把 rdb 文件以 socket 的方式发给 slave，disk 方式的时候，当一个 rdb 保存的过程中，多个 slave 都能共享这个 rdb 文件，socket 的方式就是一个个 slave顺序复制，只有在磁盘速度缓慢但是网络相对较快的情况下才使用 socket 方式，否则使用默认的 disk方式
repl-diskless-sync-delay 30 #diskless 复制的延迟时间，设置 0 为关闭，一旦复制开始还没有结束之前，
master 节点不会再接收新 slave 的复制请求，直到下一次开始
repl-ping-slave-period 10 #slave 根据 master 指定的时间进行周期性的 PING 监测
repl-timeout 60 #复制链接超时时间，需要大于 repl-ping-slave-period，否则会经常报超时
repl-disable-tcp-nodelay no #在 socket 模式下是否在 slave 套接字发送 SYNC 之后禁用 TCP_NODELAY，如果你选择“yes”Redis 将使用更少的 TCP 包和带宽来向 slaves 发送数据。但是这将使数据传输到 slave上有延迟，Linux 内核的默认配置会达到 40 毫秒，如果你选择了 "no" 数据传输到 salve 的延迟将会减少但要使用更多的带宽
repl-backlog-size 1mb #复制缓冲区大小，只有在 slave 连接之后才分配内存。
repl-backlog-ttl 3600 #多次时间 master 没有 slave 连接，就清空 backlog 缓冲区。
replica-priority 100 #当 master 不可用，Sentinel 会根据 slave 的优先级选举一个 master。最低的优先级的 slave，当选 master。而配置成 0，永远不会被选举。

#设置常用的配置
requirepass foobared #设置 redis 连接密码
rename-command #重命名一些高危命令
maxclients 10000 #最大连接客户端
maxmemory #最大内存，单位为 bytes 字节，8G 内存的计算方式 8(G)*1024(MB)*1024(KB)*1024(Kbyte)，需要注意的是 slave 的输出缓冲区是不计算在 maxmemory 内。
appendonly no #是否开启 AOF 日志记录，默认 redis 使用的是 rdb 方式持久化，这种方式在许多应用中已经足够用了。但是 redis 如果中途宕机，会导致可能有几分钟的数据丢失，根据 save 来策略进行持久化，Append Only File 是另一种持久化方式，可以提供更好的持久化特性。Redis 会把每次写入的数据在接收后都写入 appendonly.aof 文件，每次启动时 Redis 都会先把这个文件的数据读入内存里，先忽略 RDB 文件。
appendfilename "appendonly.aof" #AOF 文件名
appendfsync everysec #aof 持久化策略的配置,no 表示不执行 fsync,由操作系统保证数据同步到磁盘,always 表示每次写入都执行 fsync，以保证数据同步到磁盘,everysec 表示每秒执行一次 fsync，可能会导致丢失这 1s 数据。
no-appendfsync-on-rewrite no #在 aof rewrite 期间,是否对 aof 新记录的 append 暂缓使用文件同步策略,主要考虑磁盘 IO 开支和请求阻塞时间。默认为 no,表示"不暂缓",新的 aof 记录仍然会被立即同步，Linux 的默认 fsync 策略是 30 秒，如果为 yes 可能丢失 30 秒数据，但由于 yes 性能较好而且会避免出现阻塞因此比较推荐。
auto-aof-rewrite-percentage 100 # 当 Aof log 增长超过指定百分比例时，重写 log file， 设置为 0 表示不自动重写 Aof 日志，重写是为了使 aof 体积保持最小，而确保保存最完整的数据。
auto-aof-rewrite-min-size 64mb #触发 aof rewrite 的最小文件大小
aof-load-truncated yes #是否加载由于其他原因导致的末尾异常的 AOF 文件(主进程被 kill/断电等)
aof-use-rdb-preamble yes #redis4.0 新增 RDB-AOF 混合持久化格式，在开启了这个功能之后，AOF 重写产生的文件将同时包含 RDB 格式的内容和 AOF 格式的内容，其中 RDB 格式的内容用于记录已有的数据，而 AOF 格式的内存则用于记录最近发生了变化的数据，这样 Redis 就可以同时兼有 RDB 持久化和AOF 持久化的优点（既能够快速地生成重写文件，也能够在出现问题时，快速地载入数据）。

lua-time-limit 5000 #lua 脚本的最大执行时间，单位为毫秒

#集群
cluster-enabled yes #是否开启集群模式，默认是单机模式
cluster-config-file nodes-6379.conf #由 node 节点自动生成的集群配置文件
cluster-node-timeout 15000 #集群中 node 节点连接超时时间
cluster-replica-validity-factor 10 #在执行故障转移的时候可能有些节点和 master 断开一段时间数据比较旧，这些节点就不适用于选举为 master，超过这个时间的就不会被进行故障转移
cluster-migration-barrier 1 #一个主节点拥有的至少正常工作的从节点，即如果主节点的 slave 节点故障后会将多余的从节点分配到当前主节点成为其新的从节点。
cluster-require-full-coverage no #集群槽位覆盖，如果一个主库宕机且没有备库就会出现集群槽位不全，那么 yes 情况下 redis 集群槽位验证不全就不再对外提供服务，而 no 则可以继续使用但是会出现查询数据查不到的情况(因为有数据丢失)。

#Slow log 是 Redis 用来记录查询执行时间的日志系统，slow log 保存在内存里面，读写速度非常快，因此你可以放心地使用它，不必担心因为开启 slow log 而损害 Redis 的速度。

slowlog-log-slower-than 10000 #以微秒为单位的慢日志记录，为负数会禁用慢日志，为 0 会记录每个命令操作。
slowlog-max-len 128 #记录多少条慢日志保存在队列，超出后会删除最早的，以此滚动删除
127.0.0.1:6379> slowlog len
(integer) 14
127.0.0.1:6379> slowlog get
1) 1) (integer) 14
 2) (integer) 1544690617
 3) (integer) 4
 4) 1) "slowlog"
127.0.0.1:6379> SLOWLOG reset
OK

```

## 3.redis数据类型，使用。

### 3.1.字符串

- 字符串是所有编程语言中最常见的和最常用的数据类型，而且也是 redis 最基本的数据类型之一，而且 redis 中所有的 key 的类型都是字符串。



```bash
#添加一个key
set key1 value1
#获取一个key的内容
get key2
#获取类型
type key1
string
# 设置过期时间
set key1 value1 ex 3  #设置自动过期时间   
#删除一个key
del key2
#批量设置多个 key：
 MSET key1 value1 key2 value2
#批量获取多个 key：
MSET key1 value1 key2 value2
#追加数据
APPEND key1 append
#数值递增：
set num 10
INCR num   #输入一次递增一次
#数值递减：
 set num 10
 DECR num   #输入一次递减一次
#返回字符串 key 长度：
 STRLEN key1

```

### 3.2列表(list)

- 列表是一个双向可读写的管道，其头部是左侧尾部是右侧，一个列表最多可以包含 2^32-1 个元素即4294967295 个元素。

```bash
#生成列表并插入数据：
 LPUSH list1 jack tom jhon
  TYPE list1    #查看类型
#向列表追加数据
LPUSH list1 tom     #左加入   左边是头部
RPUSH list1 jack   #右加入   右边是尾部
# 获取列表长度
 LLEN list1
#移除列表数据
RPOP list1 #最后一个
LPOP list1 #第一个

```



### 3.3集合（set）

- Set 是 String 类型的无序集合。集合成员是唯一的，这就意味着集合中不能出现重复的数据。

```bash
#生成集合 key
 SADD set1 v1 
 SADD set2 v2 v3  #生成多个集合
#查看类型
 TYPE set1
#追加数值
追加的时候不能追加已经存在的数值
SADD set1 v2 v3 v4  
#查看集合的所有数据
 SMEMBERS set1
#获取集合的差集
#获取集合的差集：
差集：已属于 A 而不属于 B 的元素称为 A 与 B 的差（集）
SDIFF set1 set2
#获取集合的交集：
交集：已属于 A 且属于 B 的元素称为 A 与 B 的交（集）
SINTER set1 set2
#获取集合的并集：
并集：已属于 A 或属于 B 的元素为称为 A 与 B 的并（集）
SUNION set1 set2
```

### 3.4sorted set(有序集合)

- Redis 有序集合和集合一样也是 string 类型元素的集合,且不允许重复的成员，不同的是每个元素都会关联一个 double(双精度浮点型)类型的分数，redis 正是通过分数来为集合中的成员进行从小到大的排序，有序集合的成员是唯一的,但分数(score)却可以重复，集合是通过哈希表实现的，所以添加，删除，查找的复杂度都是 O(1)， 集合中最大的成员数为 2^32 - 1 (4294967295, 每个集合可存储 40 多亿个成员)。

```bash
#生成有序集合
 ZADD zset1 1 v1
 ZADD zset1 2 v2
 ZADD zset1 2 v3
 ZADD zset1 3 v4
#查看你类型
TYPE zset1

#排行案例：
192.168.7.104:6379> ZADD paihangbang 10 key1 20 key2 30 key3
(integer) 3
192.168.7.104:6379> ZREVRANGE paihangbang 0 -1 withscores #显示指定集合内所有 key 和得分情况
1) "key3"
2) "30"
3) "key2"
4) "20"
5) "key1"
6) "10"


#批量添加多个数值：
ZADD zset2 1 v1 2 v2 4 v3 5 v5
#获取集合的长度数
ZCARD zset1
ZCARD zset2
#基于索引返回数值：
ZRANGE zset1 1 3
1) "v2"
2) "v3"
3) "v4"
127.0.0.1:6379> ZRANGE zset1 0 2
1) "v1"
2) "v2"
3) "v3"
127.0.0.1:6379> ZRANGE zset1 2 2
1) "v3"
#返回某个数值的索引：
127.0.0.1:6379> ZRANK zset1 v2
(integer) 1
127.0.0.1:6379> ZRANK zset1 v3
(integer) 2
```



### 3.5哈希(hash)

- hash 是一个 string 类型的 field 和 value 的映射表，hash 特别适合用于存储对象,Redis 中每个 hash 可以存储 232 - 1 键值对（40 多亿）。

```bash
#生成 hash key：
127.0.0.1:6379> HSET hset1 name tom age 18
(integer) 1
127.0.0.1:6379> TYPE hset1
hash
#获取 hash key 字段值：
127.0.0.1:6379> HGET hset1 name
"tom"
127.0.0.1:6379> HGET hset1 age
"18"
#删除一个 hash key 的字段：
127.0.0.1:6379> HDEL hset1 age
(integer) 1
#获取所有 hash 表中的字段：
127.0.0.1:6379> HSET hset1 name tom age 19
(integer) 1
127.0.0.1:6379> HKEYS hset1
1) "name"
2) "age"

```

### 3.6消息队列：

- 消息队列主要分为两种，分别是生产者消费者模式和发布者订阅者模式，这两种模式 Redis 都支持.

- 队列当中的 消息由不同的生产者写入也会有不同的消费者取出进行消费处理，但是一个消息一定是只能被取出一次也就是被消费一次。

```bash
#生产者发布消息：
[root@redis-s4 ~]# redis-cli
127.0.0.1:6379> AUTH 123456
OK
127.0.0.1:6379> LPUSH channel1 msg1 #从管道的左侧写入
(integer) 1
127.0.0.1:6379> LPUSH channel1 msg2
(integer) 2
127.0.0.1:6379> LPUSH channel1 msg3
(integer) 3
127.0.0.1:6379> LPUSH channel1 msg4
(integer) 4
127.0.0.1:6379> LPUSH channel1 msg5
(integer) 5
#查看队列所有消息：
127.0.0.1:6379> LRANGE channel1 0 -1
1) "msg5"
2) "msg4"
3) "msg3"
4) "msg2"
5) "msg1"
#消费者消费消息：
127.0.0.1:6379> RPOP channel1 #从管道的右侧消费
"msg1"
127.0.0.1:6379> RPOP channel1
"msg2"
127.0.0.1:6379> RPOP channel1
"msg3"
127.0.0.1:6379> RPOP channel1
"msg4"
127.0.0.1:6379> RPOP channel1
"msg5"
127.0.0.1:6379> RPOP channel1
(nil)
#再次验证队列消息：
127.0.0.1:6379> LRANGE channel1 0 -1
(empty list or set) #队列中的消息已经被已全部消费完毕
```

### 3.7发布者订阅模式

- 在发布者订阅者模式下，发布者将消息发布到指定的 channel 里面，凡是监听该 channel 的消费者都会收到同样的一份消息，这种模式类似于是收音机模式，即凡是收听某个频道的听众都会收到主持人发布的相同的消息内容。
  此模式常用语群聊天、群通知、群公告等场景。
  	Subscriber：订阅者
  	Publisher：发布者
  	Channel：频道



```bash
#订阅者监听频道：
[root@redis-s4 ~]# redis-cli
127.0.0.1:6379> AUTH 123456
OK
127.0.0.1:6379> SUBSCRIBE channel1 #订阅者订阅指定的频道
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "channel1"
3) (integer) 1
#发布者发布消息：
127.0.0.1:6379> PUBLISH channel1 test1 #发布者发布消息
(integer) 2
127.0.0.1:6379> PUBLISH channel1 test2
(integer) 2
127.0.0.1:6379>
#订阅多个频道：
#订阅指定的多个频道
SUBSCRIBE channel1 channel2
#订阅所有频道：
PSUBSCRIBE *
#订阅匹配的频道：
PSUBSCRIBE chann* #匹配订阅多个频道
```

### 3.8redis的其他命令

#### 3.8.1  config

```bash
#config 命令用于查看当前 redis 配置、以及不重启更改 redis 配置等
#更改最大内存：
127.0.0.1:6379> CONFIG set maxmemory 8589934592
OK
127.0.0.1:6379> CONFIG get maxmemory #查看特定的配置，获取 * 所有的
1) "maxmemory"
2) "8589934592"

#设置连接密码：
127.0.0.1:6379> CONFIG SET requirepass 123456
OK
```

#### 3.8.2  info

```bash
#显示当前节点 redis 运行状态信息
127.0.0.1:6379> info
...

...
```

#### 3.8.3 keys

```bash
#查看当前库下的所有 key：
127.0.0.1:6379> key *
...
...

```

#### 3.8.4 bgsave

```bash
#手动在后台执行 RDB 持久化操作
127.0.0.1:6379>  bgsave
Background saving started
```

#### 3.8.5 dbsize

```bash
#返回当前库下的所有 key 数量
127.0.0.1:6379> dbsize
```

#### 3.8.6 flushdb

```bash
#强制清空当前库中的所有 key
(危险操作，一般都不用)
```

#### 3.8.7 flushall

```bash
#强制清空当前 redis 服务器所有数据库中的所有 key，即删除所有数据

删库了（你开心就好_-_）
```

## 4 redis 高可用与集群

- 虽然 Redis 可以实现单机的数据持久化，但无论是 RDB 也好或者 AOF 也好，都解决不了点宕机问题，即一旦redis 服务器本身出现系统故障、硬件故障等问题后，就会直接造成数据的丢失，因此需要使用另外的技术来解决单点问题。

### 4.1 配置redis 主从

- 主备模式，可以实现 Redis 数据的跨主机备份。
- 程序端连接到高可用负载的 VIP，然后连接到负载服务器设置的 Redis 后端 real server，此模式不需要在程序里面配置 Redis 服务器的真实 IP 地址，当后期 Redis 服务器 IP 地址发生变更只需要更改 redis相应的后端 real server 即可，可避免更改程序中的 IP 地址设置

![1560381350089](C:\Users\L\AppData\Roaming\Typora\typora-user-images\1560381350089.png)



#### 4.1.1 Slave 主要配置：

- **Redis Slave 也要开启持久化并设置和 master 同样的连接密码，因为后期 slave 会有提升为 master 的可能,Slave 端切换 master 同步后会丢失之前的所有数据。**





































































