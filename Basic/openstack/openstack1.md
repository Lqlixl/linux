# openstack搭建环境



#### 时间同步

```bash
 
 cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
 
 #时间同步命令
 ntpdate 172.20.0.1
 
 #任务计划
 [root@centos ~]# crontab -e
*/30 * * * * /usr/sbin/ntpdate 172.20.0.1 && /usr/sbin/hwclock -w
[root@centos ~]# crontab -l


[root@centos ~]# echo "*/30 * * * * /usr/sbin/ntpdate 172.20.0.1 && /usr/sbin/hwclock -w" > /var/spool/cron/root
 

#重启服务
[root@centos ~]# systemctl restart crond.service
```

### 配置yum源

```bash
 #各服务器重新配置 yum 源
  yum install wget –y
  rm -rf /etc/yum.repos.d/*
 wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
  wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel7.repo

```



## 2. 环境搭建

### 2.1 搭建管理端

```bash
#搭建网络
7.2版本以前需要安装，才能进行多网卡绑定
[root@centos ~]# yum install bridge-utils -y

#绑定网卡

# bond0 
[root@centos network-scripts]# cat ifcfg-bond0
BOOTPROTO=static
NAME=bond0
DEVICE=bond0
ONBOOT=yes
BONDING_MASTER=yes
BONDING_OPTS="mode=1 miimon=100"
BRIDGE=br0

#bond
[root@centos network-scripts]# cat ifcfg-bond1
BOOTPROTO=static
NAME=bond1
DEVICE=bond1
ONBOOT=yes
BONDING_MASTER=yes
BONDING_OPTS="mode=1 miimon=100"
BRIDGE=br1

#br1 仅主机
[root@centos network-scripts]# cat ifcfg-br1
TYPE=Bridge
BOOTPROTO=static
NAME=br1
DEVICE=br1
ONBOOT=yes
IPADDR=192.168.1.7

# br0 桥接
[root@centos network-scripts]# cat ifcfg-br0
TYPE=Bridge
BOOTPROTO=static
NAME=br0
DEVICE=br0
ONBOOT=yes
IPADDR=172.20.76.7
NETMASK=255.255.255.0
GATEWAY=172.20.76.254
DNS1=114.114.114.144
DNS2=223.5.5.5

#eth0
[root@centos network-scripts]# cat ifcfg-eth0
TYPE=Ethernet
BOOTPROTO=static
NAME=eth0
DEVICE=eth0
ONBOOT=yes
NM_CONTROLLED=no
MASTER=bond1
USERCTL=no
SLAVE=yes

#eth1
[root@centos network-scripts]# cat ifcfg-eth1
TYPE=Ethernet
BOOTPROTO=static
NAME=eth1
DEVICE=eth1
ONBOOT=yes
NM_CONTROLLED=no
MASTER=bond0
USERCTL=no
SLAVE=yes

#eth2
[root@centos network-scripts]# cat ifcfg-eth2
TYPE=Ethernet
BOOTPROTO=static
NAME=eth2
DEVICE=eth2
ONBOOT=yes
NM_CONTROLLED=no
MASTER=bond0
USERCTL=no
SLAVE=yes

#eth3
[root@centos network-scripts]# cat ifcfg-eth3
TYPE=Ethernet
BOOTPROTO=static
NAME=eth3
DEVICE=eth3
ONBOOT=yes
NM_CONTROLLED=no
MASTER=bond0
USERCTL=no
SLAVE=yes

```

#### 2.1.1 修改主机名

```bash
[root@centos ~]# vim /etc/hostname
openstack.lqlixl.net

```





### 2.2搭建keepalive+haproxy

## 2.0 keepalive,haproxy配置后面再配。

#### 2.2.1 配置网络

```bash

#桥接网卡
lqlixl@ubuntu1804:~$ cat /etc/netplan/01-netcfg.yaml 
# This file describes the network interfaces available on your system
# For more information, see netplan(5).
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: no
      dhcp6: no
      addresses: [172.20.76.140/24]
      gateway4: 172.20.76.254
      nameservers:
        addresses: [223.5.5.5]

#仅主机
lqlixl@ubuntu1804:~$ cat /etc/netplan/02-netcfg.yaml 
# This file describes the network interfaces available on your system
# For more information, see netplan(5).
network:
  version: 2
  renderer: networkd
  ethernets:
    eth1:
      dhcp4: no
      dhcp6: no
      addresses: [192.168.1.140/24]

#重启,应用
neplan apply
```

#### 2.2.2 配置yum

```bash
# 阿里源
root@ubuntu1804:/home/zhuzhuzhu# vim /etc/apt/sources.list

deb http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse

#更新源
root@ubuntu1804:/home/lqlixl# apt-get update


```

#### 2.2.3 keepalived+haproxy

```bash
#安装keepalived+haproxy
root@ubuntu1804:/home/lqlixl# apt-get install haproxy keepalived

...

#启动服务，并设置开机启动
systemctl start keepalived && systemctl enable keepalived
systemctl start haproxy &&systemctl enable haproxy


#寻找keepalived 的配置文件
root@lb:/home/lqlixl# find / -name keepalived*
/usr/share/doc/keepalived/samples/keepalived.conf.sample


#复制配置文件
cp /usr/share/doc/keepalived/samples/keepalived.conf.sample /etc/keepalived/keepalived.conf


#配置配置文件
root@lb:/home/lqlixl# cat /etc/keepalived/keepalived.conf
! Configuration File for keepalived

global_defs {
   notification_email {
     acassen
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 192.168.200.1
   smtp_connect_timeout 30
   router_id LVS_DEVEL
}

vrrp_instance VI_1 {
    interface eth0
    virtual_router_id 50
    priority 100
    advert_int 1
    virtual_ipaddress {
        172.20.76.10 dev eth0 label eth0:0
  
    }
}


#重启服务
root@lb:/home/zhuzhuzhu# systemctl restart keepalived

#查看网络是否有vip
root@lb:/home/zhuzhuzhu# ifconfig

#ping VIP
root@lb:/home/zhuzhuzhu# ping 172.20.76.10

```

#### 2.2.3.1 搭建主从

```bash
#配置配置文件
root@lb2:/home/lqlixl# cat /etc/keepalived/keepalived.conf
! Configuration File for keepalived

global_defs {
   notification_email {
     acassen
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 192.168.200.1
   smtp_connect_timeout 30
   router_id LVS_DEVEL
}

vrrp_instance VI_1 {
    interface eth0
    virtual_router_id 50
    priority 80
    advert_int 1
    virtual_ipaddress {
        172.20.76.140 dev eth0 label eth0:0
  
    }
}
```

#### 2.2.3.2 验证

```bash
#master 停服务
systemctl stop keepalived

#查看你VIP飘到备机器上没有
```



#### 2.2.4 修改主机名

```bash
#修改主机名
root@ubuntu1804:/home/lqlixl# vim /etc/hostname 
lb.lqlixl.net
```

## 3. openstack

#### 3.1安装和配置源

```bash
#查看openstack版本
yum list centos-release-openstack*

#各服务器配置yum源
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
wget -O /
etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel7.repo

#各服务器安装 ocata 的 yum 源：
 yum install –y centos-release-openstack-ocata.noarch
# yum install -y https://rdoproject.org/repos/rdo-release.rpm
 
 #各服务器安装 openstack 客户端和屏蔽selinux
  yum install python-openstackclient openstack-selinux -y

#mysql 和控制端不在一个服务器需要安装的包
#用于控制端连接数据库
yum install python2-PyMySQL.noarch -y

#memcache 和控制端不在一个服务器需要安装的包
#用于控制memcache服务端
yum install python-memcached.noarch -y


```

- 修改

```bash
#临时改成S版  再重装openstack dashboard就可用了rm -rf
要是安装rdo  包
[centos-openstack-ocata]
name=CentOS-7 - OpenStack ocata
#baseurl=http://mirror.centos.org/centos/7/cloud/$basearch/openstack-ocata/
#baseurl=https://mirrors.aliyun.com/centos/7/cloud/x86_64/openstack-ocata/
baseurl=https://mirrors.aliyun.com/centos/7/cloud/x86_64/openstack-stein/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-Cloud
exclude=sip,PyQt4

```



## 4.mysql+rabbit+memcache

### 4.1 mysql

```bash
#安装mariadb
yum install mariadb mariadb-server -y

#启动服务，开机启动
systemctl start mariadb.service  && systemctl enable mariadb.service 

ss -ntl
```

#### 4.1.2 配置数据库

```bash
#配置数据库
[root@centos ~]# cat /etc/my.cnf.d/openstack.cnf 
[mysqld]
bind-address = 0.0.0.0#指定监听地址
default-storage-engine = innodb #默认引擎
innodb_file_per_table = on #开启每个表都有独立表空间
max_connections = 4096 #最大连接数
collation-server = utf8_general_ci #不区分大小写排序
character-set-server = utf8 #设置编码
```

#### 4.1.4 跑安全脚本

```bash
#安全脚本
[root@centos ~]# mysql_secure_installation 
y
redhat
redhat
y
"login remotely"  n
y


#查看生效没有
[root@centos ~]# mysql -uroot -predhat

```

#### 4.1.3 配置my.cnf

```bash
#主配置文件
[root@centos data]# cat /etc/my.cnf
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
user=mysql
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
datadir=/data/mysql
innodb_file_per_table=1
#skip-grant-table=1
relay-log =/data/mysql
server-id=1
log-error=/data/mysql=log/mysql_error.txt
log-bin=/data/mysql-binlog/master-log
#general_log=ON
#general_log_file=/data/general_mysql.log
long_query_time=5
slow_query_log=1
slow_query_log_file= /data/mysql-log/slow_mysql.txt
max_connections=1000
bind-address=192.168.10.204
[client]
port=3306
socket=/var/lib/mysql/mysql.sock
[mysqld_safe]
log-error=/data/mysql-log/mysqld-safe.log
pid-file=/var/lib/mysql/mysql.sock

#创建目录
[root@centos data]# mkdir /data/{mysql,mysql-binlog,mysql-log...} -pv

#授权
chown mysql.mysql /data/

#重启
 systemctl restart mariadb
 
 
```

#### 4.1.4 修改主机名

```bash
vim /etc/hostname
mysql.lqlixl.net

```

#### 4.1.5 解析域名

```bash
vim /etc/hosts
172.20.76.27  msyql
要是设置了从数据库也要写
```



### 4.2 rabbitMQ 

#### 4.2.1安装 rabbitMQ 服务器

- 各组件通过消息发送与接收是实现组件之间的通信

```bash
#安装rabbitmq
[root@centos ~]# yum install rabbitmq-server -y

#启动并设置为开机启动
systemctl start rabbitmq-server.service && systemctl enable rabbitmq-server.service

```

#### 4.2.2 配置登录信息等

```bash
#添加 rabbitMQ 客户端用户并设置密码：
rabbitmqctl add_user openstack 123456

#赋予 openstack 用户读写权限：
rabbitmqctl set_permissions openstack ".*" ".*" ".*"

#打开 rabbitMQ 的 web 插件：
rabbitmq-plugins enable rabbitmq_management

#查看插件
rabbitmq-plugins list 

#访问 rabbitMQ 的 web 界面：
 默认用户名密码都是 guest，可以更改，web 访问端口为 15672
 
 #验证
ss -ntl
State       Recv-Q Send-Q Local Address:Port               Peer Address:Port              
LISTEN      0      128        *:25672                   *:*                  
LISTEN      0      50         *:3306                   *:*                  
LISTEN      0      128        *:4369                   *:*                  
LISTEN      0      128        *:22                     *:*                  
LISTEN      0      100    127.0.0.1:25                     *:*                  
LISTEN      0      128       :::5672                  :::*                  
LISTEN      0      128       :::22                    :::*                  
LISTEN      0      100      ::1:25                    :::* 


#客服端使用    5672
#web访问端口     15672
#集群通信使用   25672


#使用192.168.1.27:15672  访问
账户：guest
密码：guest

```

#### 4.2.3 制作高可用性

- <http://blogs.studylinux.net/?p=4266>
- 前提条件
  - cookie 一致
  - 能不能释放对方的主机名，不包含域名主机名，比如mysql.lqlixl.net 中的msyql

```bash
#设置相同的cookie值，随便复制那个服务器里的cookie都可以
cat /var/lib/rabbitmq/.erlang.cookie 
RDQSENKVOCONXAXPBIDY

#复制cookie到其他服务器
cp /var/lib/rabbitmq/.erlang.cookie 192.168.1.28://var/lib/rabbitmq/

#停止应程序
 rabbitmqctl  stop_app 
 
 #清空元数据（slave）
rabbitmqctl   reset 

#未添加到集群之前只有自己一个节点，单机状态
rabbitmqctl  cluster_status 

#将rabbitmq-server1添加到集群当中，并成为内存节点，不加--ram默认是磁盘节点
rabbitmqctl  join_cluster rabbit@rabbitmq-server2 --ram　

#更改为镜像模式（互为主主一类）
rabbitmqctl set_policy  ha-all "#"  '{"ha-mode":"all"}' 

```





### 4.3 Memcache

- 用于缓存 openstack 各服务的身份认证令牌信息,

- 纯粹的seesion信息

- memcache 和控制端不在一个服务器需要安装的包，控制端需要安装一个包，控制端控制memcache

  > yum install python-memcached.noarch -y

#### 4.3 .1 安装和配置文件

```bash
#安装
yum install memcached

#启动服务和开机启动
 systemctl start memcached && systemctl enable memcached

```

#### 4.3.2 配置文件

```bash
#修改配置文件
cat /etc/sysconfig/memcached
PORT="11212" #避免和 haproxy 监听的 11211 冲突
USER="memcached"
MAXCONN="1024"
CACHESIZE="512"
OPTIONS="-l 0.0.0.0,::1"

#重启服务
 systemctl restart memcached

```

#### 4.3.2 验证

```bash
[root@centos ~]# ss -ntl
State      Recv-Q Send-Q        Local Address:Port                       Peer Address:Port              
LISTEN     0      128                       *:25672                                 *:*                  
LISTEN     0      869                       *:3306                                  *:*                  
LISTEN     0      1024                      *:11211                                 *:*                  
LISTEN     0      128                       *:4369                                  *:*                  
LISTEN     0      128                       *:22                                    *:*                  
LISTEN     0      128                       *:15672                                 *:*                  
LISTEN     0      100               127.0.0.1:25                                    *:*                  
LISTEN     0      128                      :::5672                                 :::*                  
LISTEN     0      1024                    ::1:11211                                :::*                  
LISTEN     0      128                      :::22                                   :::*                  
LISTEN     0      100                     ::1:25                                   :::*         


#memcache监听端口：11211
```

## 5 haproxy 配置

### 5.1 配置主配置文件

```bash
root@ubuntu1804:~# cat /etc/haproxy/haproxy.cfg 

global
maxconn 100000
#chroot /usr/local/haproxy
#stats socket /var/lib/haproxy/haproxy.sock mode 600 level admin
uid 99
gid 99
daemon
nbproc 4
cpu-map 1 0
cpu-map 2 1
cpu-map 3 2
cpu-map 4 3
#pidfile /usr/local/haproxy/run/haproxy.pid
log 127.0.0.1 local3 info

defaults
option http-keep-alive
option  forwardfor
maxconn 100000
mode http
timeout connect 300000ms
timeout client  300000ms
timeout server  300000ms

listen stats
 mode http
 bind 0.0.0.0:9999
 stats enable
 log global
 stats uri     /haproxy-status
 stats auth    haadmin:q1w2e3r4ys

#监听数据库
listen openstack_mysql_port     
 bind 172.20.76.10:3306
 mode tcp
 log global
 server 172.20.76.10  172.20.76.10:3306  check inter 3000 fall 2 rise 5

#监听消息队列
listen openstack_rabbit_port
 bind 172.20.76.10:5672
 mode tcp
 log global
 server 172.20.76.10  172.20.76.10:5672  check inter 3000 fall 2 rise 5

#监听缓存
listen openstack_memcache_port
 bind 172.20.76.10:11211
 mode tcp
 log global
 server 172.20.76.10  172.20.76.10:11211  check inter 3000 fall 2 rise 5


#重启服务
systemctl restart haproxy
```

### 5.2 查看端口

```bash
root@ubuntu1804:~# ss -ntl
State     Recv-Q     Send-Q            Local Address:Port            Peer Address:Port     
LISTEN    0          128                172.20.76.10:5672                 0.0.0.0:*        
LISTEN    0          128                172.20.76.10:3306                 0.0.0.0:*        
LISTEN    0          128                172.20.76.10:11211                0.0.0.0:*        
LISTEN    0          128                     0.0.0.0:9999                 0.0.0.0:*        
LISTEN    0          128               127.0.0.53%lo:53                   0.0.0.0:*        
LISTEN    0          128                     0.0.0.0:22                   0.0.0.0:*        
LISTEN    0          128                   127.0.0.1:6010                 0.0.0.0:*        
LISTEN    0          128                   127.0.0.1:6011                 0.0.0.0:*        
LISTEN    0          128                        [::]:22                      [::]:*        
LISTEN    0          128                       [::1]:6010                    [::]:*        
LISTEN    0          128                       [::1]:6011                    [::]:*


#监听信息

 172.20.76.10:5672    消息队列
 172.20.76.10:3306   数据库
 172.20.76.10:11211  缓存

```

### 5.3 测试通不通

```bash
#在控制端安装测试工具
yum install telnet -y

#测试消息队列
telnet 172.20.76.10 5672

#测试数据库
telnet 172.20.76.10 3306    没有认证，不能通。

#测试缓存
telnet 172.20.76.10 11211

192.168.1.10
```































































































































































































































































































































































































































































































































































