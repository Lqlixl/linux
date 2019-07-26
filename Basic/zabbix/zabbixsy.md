## 1. 主动模式

环境

|     环境      |      IP       |
| :-----------: | :-----------: |
| zabbix server | 192.168.1.150 |
|     mysql     | 192.168.1.151 |
| zabbix proxy  | 192.168.1.152 |
| zabbix agent1 | 192.168.1.153 |
| zabbix agent2 | 192.168.1.154 |



规划图

![1563454175371](D:\学习资料\markdow\Basic\zabbix\zabbixsy.assets\1563454175371.png)



### 1.环境配置

```bash
在150 上装zabbix servere
在151 上装MySQL
详情见zabbix.md

```

### 2. 配置proxy数据库

```bash
# 192.168.1.151 mysql
mysql
create database zabbix_proxy character set utf8 collate utf8_bin;
grant all privileges on zabbix_proxy.* to zabbix_proxy@'192.168.1.%' identified by '123456';

#192.168.1.150
通过150连接测试能不能连接数据库
mysql -uzabbix_proxy -p123456 -h192.168.1.151

```

### 3.安装zabbix_proxy

```bash
# 192.168.1.152 zabbix_proxy
#配置环境
wget https://repo.zabbix.com/zabbix/4.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_4.0-2+bionic_all.deb

dpkg -i zabbix-release_4.0-2+bionic_all.deb

apt update

#安装
apt install zabbix-proxy-mysql -y

#初始化数据库
zcat /usr/share/doc/zabbix-proxy-mysql/schema.sql.gz |mysql -uzabbix_proxy -p123456 -h192.168.1.151 zabbix_proxy

```



### 4.编辑zabbix_proxy 的配置文件

```bash
root@proxy:~# vim /etc/zabbix/zabbix_proxy.conf 
#配置代理的模式0为主动1为被动
ProxyMode=0
#配置zabbix的server端地址
Server=192.168.27.10
#配置Hostname需要和图形端的代理名称相同
Hostname=Zabbix_proxy_active
#配置监听端口
ListenPort=10051
#配置SourceIp，当存在多块网卡时，指定从哪个地址发送数据
SourceIP=192.168.27.11
#开启远程命令，允许server到proxy上执行命令，在故障自愈时使用
EnableRemoteCommands=1
#数据库地址
DBHost=192.168.27.12
#数据库名
DBName=zabbix_proxy
#配置数据库用户
DBUser=zabbix_proxy
#配置数据库连接密码
DBPassword=123456
#proxy将数据发送给server后将数据在本地保存多少时间
ProxyLocalBuffer=720
#当proxy和server无法连接时将数据在本地保存多长时间，一般为720小时
ProxyOfflineBuffer=720
#用来检测server端和proxy的心跳信息，一般设置为5分钟
HeartbeatFrequency=120
#每间隔多少时间到server获取监控项，防止监控项更新后agent端无法获取到，一般设置为5分钟
ConfigFrequency=600
#数据发送的间隔时间
DataSenderFrequency=60
#开启多少个数据收集器
StartPollers=20
#配置javagateway
JavaGateway=192.168.1.153
#配置javagateway端口
JavaGatewayPort=10052
#启动多少java的数据收集器
StartJavaPollers=10
#当主机数量很多时，会将获取的监控项存放在缓存中，生产中设置2G
CacheSize=128M
#启动多少个线程和数据库连接
StartDBSyncers=10
#保存agent发送过来的监控数据的内存空间大小，生产中设置2G 
HistoryCacheSize=128M
#历史数据的索引
HistoryIndexCacheSize=32M
#获取数据的最长等待时间，30秒
Timeout=30
#配置fping用于探测网络设备
FpingLocation=/usr/bin/fping
#设置启动用户为zabbix
User=zabbix
#配置监听地址
ListenIP=0.0.0.0

```

- 详情配置

```bash

root@zabbix_proxy:~# grep "^[a-Z]" /etc/zabbix/zabbix_proxy.conf
ProxyMode=0
Server=192.168.1.150
Hostname=Zabbix_proxy
ListenPort=10051
SourceIP=192.168.1.152
LogFile=/var/log/zabbix/zabbix_proxy.log
LogFileSize=0
EnableRemoteCommands=1
PidFile=/var/run/zabbix/zabbix_proxy.pid
SocketDir=/var/run/zabbix
DBHost=192.168.1.151
DBName=zabbix_proxy
DBUser=zabbix_proxy
DBPassword=123456
ProxyLocalBuffer=720
ProxyOfflineBuffer=720
HeartbeatFrequency=120
ConfigFrequency=600
DataSenderFrequency=60
StartPollers=20
JavaGateway=192.168.1.153
JavaGatewayPort=10052
StartJavaPollers=10
SNMPTrapperFile=/var/log/snmptrap/snmptrap.log
ListenIP=0.0.0.0
CacheSize=128M
StartDBSyncers=10
HistoryCacheSize=128M
HistoryIndexCacheSize=32M
Timeout=30
ExternalScripts=/usr/lib/zabbix/externalscripts
FpingLocation=/usr/bin/fping
Fping6Location=/usr/bin/fping6
LogSlowQueries=3000

```

### 5启动zabbix_proxy

```bash
#启动服务并设置开机自启
systemctl start zabbix-proxy.service 
systemctl enable zabbix-proxy.service 

```

### 6.配置agent端

```bash
#192.168.1.153 192.168.1.154 agent1 agent2
wget https://repo.zabbix.com/zabbix/4.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_4.0-2+bionic_all.deb

dpkg -i zabbix-release_4.0-2+bionic_all.deb

apt update

#安装agent
apt install zabbix-agent -y
```

### 7.修改agent配置文件

```bash
cat /etc/zabbix/zabbix_agentd.conf 
#配置server分别指向zabbix-server和zabbix-proxy
Server=192.168.27.10,192.168.27.11
#配置ServerActive为proxy地址
ServerActive=192.168.27.11
#配置Hostname为本机地址
Hostname=192.168.27.12

#详情配置

root@agent2:~# grep "^[a-Z]" /etc/zabbix/zabbix_agentd.conf 
PidFile=/var/run/zabbix/zabbix_agentd.pid
LogFile=/var/log/zabbix/zabbix_agentd.log
LogFileSize=0
Server=192.168.1.152,192.168.1.150
ListenPort=10050
ServerActive=192.168.1.152
Hostname=192.168.1.154
Include=/etc/zabbix/zabbix_agentd.d/*.confvim /etc/zabbix/zabbix_agentd.conf 
#配置server分别指向zabbix-server和zabbix-proxy
Server=192.168.27.10,192.168.27.11
#配置ServerActive为proxy地址
ServerActive=192.168.27.11
#配置Hostname为本机地址
Hostname=192.168.27.12

```

### 8.启动zabbix-agent

```bash
systemctl start zabbix-agent
systemctl enable zabbix-agent
```



### 9.web端添加代理

## 在web上添加代理

![proxy1.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9A%84%E4%B8%BB%E5%8A%A8proxy%E6%A8%A1%E5%BC%8F/proxy1.png)
![proxy2.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9A%84%E4%B8%BB%E5%8A%A8proxy%E6%A8%A1%E5%BC%8F/proxy2.png)
![proxy3.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9A%84%E4%B8%BB%E5%8A%A8proxy%E6%A8%A1%E5%BC%8F/proxy3.png)

## 更改模板的发现规则

![discover1.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9A%84%E4%B8%BB%E5%8A%A8proxy%E6%A8%A1%E5%BC%8F/discover1.png)
分别将以下两个发现规则改为主动
![discover2.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9A%84%E4%B8%BB%E5%8A%A8proxy%E6%A8%A1%E5%BC%8F/discover2.png)
![discover3.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9A%84%E4%B8%BB%E5%8A%A8proxy%E6%A8%A1%E5%BC%8F/discover3.png)

## 更改监控项原型

将发现规则内的所有监控项原型改为主动模式
![item1.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9A%84%E4%B8%BB%E5%8A%A8proxy%E6%A8%A1%E5%BC%8F/item1.png)
![item2.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9A%84%E4%B8%BB%E5%8A%A8proxy%E6%A8%A1%E5%BC%8F/item2.png)
![item3.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9A%84%E4%B8%BB%E5%8A%A8proxy%E6%A8%A1%E5%BC%8F/item3.png)

## 在web上添加主机

![addhost1.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9A%84%E4%B8%BB%E5%8A%A8proxy%E6%A8%A1%E5%BC%8F/addhost1.png)
![addhost2.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9A%84%E4%B8%BB%E5%8A%A8proxy%E6%A8%A1%E5%BC%8F/addhost2.png)
![addhost3.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9A%84%E4%B8%BB%E5%8A%A8proxy%E6%A8%A1%E5%BC%8F/addhost3.png)![addhost4.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9A%84%E4%B8%BB%E5%8A%A8proxy%E6%A8%A1%E5%BC%8F/addhost4.png)







## 监控Tomcat

### 1.安装jdk

```bash
# 192.168.1.154
root@node5:~# apt-get install openjdk-8-jdk

root@node5:~# java -version
openjdk version "1.8.0_212"
OpenJDK Runtime Environment (build 1.8.0_212-8u212-b03-0ubuntu1.18.04.1-b03)
OpenJDK 64-Bit Server VM (build 25.212-b03, mixed mode)

```



### 2.安装tomcat

```bash
#下载tomcat
wget https://www.apache.org/dist/tomcat/tomcat-8/v8.5.43/bin/apache-tomcat-8.5.43.tar.gz

tar xf apache-tomcat-8.5.43.tar.gz -C /usr/local/

cd /usr/local/
ln -sv apache-tomcat-8.5.43/ tomcat

#启动tomcat
/usr/local/tomcat/bin/catalina.sh start

```

### 3.添加PATH环境

```bash
root@tomcat:~# vim /etc/profile.d/tomcat.sh
#!/bin/bash
export CATALINA_HOME=/usr/local/tomcat
export PATH=$CATALINA_HOME/bin:$PATH

```



### 4. 配置tomcat监控参数

```bash
cat /usr/local/tomcat/bin/catalina.sh
...另起一段
CATALINA_OPTS="$CATALINA_OPTS 
-Dcom.sun.management.jmxremote 
-Dcom.sun.management.jmxremote.port=12345
-Dcom.sun.management.jmxremote.authenticate=false
-Dcom.sun.management.jmxremote.ssl=false 
-Djava.rmi.server.hostname=192.168.27.154"
...

root@node5:~# vim /usr/local/tomcat/bin/catalina.sh  #添加配置
CATALINA_OPTS="$CATALINA_OPTS
-Dcom.sun.management.jmxremote #启用远程监控JMX
-Dcom.sun.management.jmxremote.port=12345 #默认启动的JMX端口号，要和zabbix添加主机时候的端口一致即
可
-Dcom.sun.management.jmxremote.authenticate=false #不使用用户名密码
-Dcom.sun.management.jmxremote.ssl=false #不使用ssl认证
-Djava.rmi.server.hostname=192.168.1.154" #tomcat主机自己的IP地址，不要写zabbix服务器的地址

#重启服务后确认端口监听
root@node5:~# ss -nlt
State Recv-Q Send-Q Local Address:Port Peer Address:Port
LISTEN 0 50 *:12345 *:*

```



### 5.配置Java gateway

#### 5.1 安装

- 192.168.1.152

```bash
root@gateway:~# wget https://repo.zabbix.com/zabbix/4.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_4.0-2+bionic_all.deb
root@gateway:~# dpkg -i zabbix-release_4.0-2+bionic_all.deb
root@gateway:~# apt update 

#在gateway主机上安装zabbix-java-gateway

root@gateway:~# apt install zabbix-java-gateway -y
```

#### 5.2 修改配置文件

```bash
root@gateway:~# grep -v "^$" /etc/zabbix/zabbix_java_gateway.conf | grep -v "#"
LISTEN_IP="0.0.0.0"
LISTEN_PORT=10052
PID_FILE="/var/run/zabbix/zabbix_java_gateway.pid"
START_POLLERS=10     #启动多少个进程轮询java，要和java 应用保持一定关系
TIMEOUT=30      #超时时间设置为30秒，

```

#### 5.3重启zabbix-java-gateway服务

```bash
root@zabbix_proxy:~# systemctl restart zabbix-java-gateway
root@zabbix_proxy:~# ss -ntl
LISTEN  0        50                       *:10052                  *:* 	

```

### 6.配置zabbix_server

```bash
root@zabbix:~# vim /etc/zabbix/zabbix_server.conf 
JavaGateway=192.168.1.152        #指定java gateway的地址
JavaGatewayPort=10052        #指定java gateway的服务器监听端口，如果是默认端口可以不
StartJavaPollers=10         #启动多少个进程去轮训 java gateway，要和java gateway的配置一致

```

###  7.在web界面上添加主机

![1563530185302](D:\学习资料\markdow\Basic\zabbix\zabbixsy.assets\1563530185302.png)



![1563530236062](D:\学习资料\markdow\Basic\zabbix\zabbixsy.assets\1563530236062.png)

![1563530247215](D:\学习资料\markdow\Basic\zabbix\zabbixsy.assets\1563530247215.png)



![1563530256705](D:\学习资料\markdow\Basic\zabbix\zabbixsy.assets\1563530256705.png)

### 8. 监控java排错方法

```bash
#测试能否获取到java 当前已经分配的 线程数，cmdline-jmxclient-0.10.3.jar此包需要自行下载
#java -jar cmdline-jmxclient-0.10.3.jar - 192.168.172.95:12345 'Catalina:name="http-nio8080",type=ThreadPool' currentThreadCount
#java -jar cmdline-jmxclient-0.10.3.jar - 192.168.172.95:12345 'Catalina:name="http-nio8080",type=ThreadPool' maxThreads


```

## 4. 监控TCP连接数

### 1.创建获取数据的脚本

```bash

oot@agent2:# cat /etc/zabbix/zabbix_agentd.d/tcp_conn.sh
#!/bin/bash
tcp_conn_status(){
        TCP_STAT=$1
        ss -ant | awk 'NR>1 {++s[$1]} END {for(k in s) print k,s[k]}' > /tmp/tcp_conn.txt
        TCP_STAT_VALUE=$(grep "$TCP_STAT" /tmp/tcp_conn.txt | cut -d ' ' -f2)
        if [ -z $TCP_STAT_VALUE ];then
                TCP_STAT_VALUE=0
        fi
        echo $TCP_STAT_VALUE
}

main(){
	case $1 in
	    tcp_status)
		tcp_conn_status $2;	
		;;
		*)
		echo "$0 + tcp_status + STATUS"
	esac
}

main $1 $2



#授权
root@agent2:~# cd /etc/zabbix/zabbix_agentd.d/
root@agent2:/etc/zabbix/zabbix_agentd.d# chmod a+x tcp_conn.sh 

```

### 2.创建conf文件引用脚本

```bash
root@node4:~# vim /etc/zabbix/zabbix_agentd.d/tcp_conn.conf
UserParameter=linux_status[*],/etc/zabbix/zabbix_agentd.conf.d/tcp_conn.sh "$1" "$2"
root@node4:~# vim /etc/zabbix/zabbix_agentd.conf
Include=/etc/zabbix/zabbix_agentd.d/*.conf
#重启agent服务
root@node4:~# systemctl restart zabbix-agent.service
#为zabbix用户授权
root@node4:~# vim /etc/sudoers
root ALL=(ALL:ALL) ALL
zabbix ALL=(ALL) NOPASSWD:ALL


```

### 3.server端测试连接数据

```bash
root@ubuntu-1804:~# zabbix_get -s 192.168.1.154 -p 10050 -k "linux_status[tcp_status,TIME-WAIT]"
5
```

### 4. 在zabbix的web界面上导入模板


![tcp2.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9B%91%E6%8E%A7tcp%E8%BF%9E%E6%8E%A5/tcp2.png)
![tcp3.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9B%91%E6%8E%A7tcp%E8%BF%9E%E6%8E%A5/tcp3.png)
![tcp4.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9B%91%E6%8E%A7tcp%E8%BF%9E%E6%8E%A5/tcp4.png)

### 5.将模板关联到主机

![add1.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9B%91%E6%8E%A7tcp%E8%BF%9E%E6%8E%A5/add1.png)
![add2.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9B%91%E6%8E%A7tcp%E8%BF%9E%E6%8E%A5/add2.png)

### 6. 验证数据已经收集

![status.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9B%91%E6%8E%A7tcp%E8%BF%9E%E6%8E%A5/status.png)



## 5. 监控memcached



### 1. 安装memcached、nc

```bash
在192.168.1.154上设置

#安装netcat linux中的瑞士军刀
apt-get install netcat* -y

#安装memcached
apt-get install memcached -y

#启动服务
systemctl start memcached

```

### 2.测试能否以非交互模式取出值

```bash
#设置memcached的连接
vim /etc/memcached.conf
...
-l 0.0.0.0
...

#测试
echo -e "stats\nquit" |nc 127.0.0.1 "11211"
STAT pid 79657
STAT uptime 577
STAT time 1563950834
STAT version 1.5.6 Ubuntu
...

OK

```

### 3.设置zabbix-agent

```bash
#编写监控memcached脚本
cat /etc/zabbix/zabbix_agentd.d/memcached.sh
#!/bin/bash
memcached_status(){
        M_PORT=$1
        M_COMMAND=$2
        echo -e  "stats\nquit" | nc  127.0.0.1 "$M_PORT" | grep "STAT $M_COMMAND" | awk '{print $3}'
}
main(){
    case $1 in
        memcached_status)
            memcached_status $2 $3
                ;;
    esac
}

main $1 $2 $3

#脚本添加执行权限
bash /etc/zabbix/zabbix_agentd.d/memcached.sh

#测试脚本
/etc/zabbix/zabbix_agentd.d/memcached.sh memcached_status 11211 uptime


#编辑agent配置文件
vim /etc/zabbix/zabbix_agentd.conf 
Server=192.168.1.150,192.168.1.152 #将server指向zabbix-server和zabbix-proxy
ServerActive=192.168.1.152    #主动模式下的server指向proxy
Hostname=192.168.1.154   #配置hostname为本地IP
Include=/etc/zabbix/zabbix_agentd.d/*.conf   #导入配置文件

#设置子配置文件
#添加监控项
vim /etc/zabbix/zabbix_agentd.d/memcached_status.conf
UserParameter=memcache_status[*],/etc/zabbix/zabbix_agentd.d/memcached.sh "$1" "$2" "$3"

#重启服务
systemctl restart zabbix-agent

```

### 4. 在server端测试能否获取数据

```bash
# 192.168.1.150

zabbix_get -s 192.168.1.154 -p 10050 -k memcache_status["memcached_status",11211,"uptime"]

```

### 5. 1在web上创建模板

按照以下方法分别创建出curr_connections、memcached_threads两个监控项
![create1.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9B%91%E6%8E%A7memcached/create1.png)
![create2.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9B%91%E6%8E%A7memcached/create2.png)
![create3.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9B%91%E6%8E%A7memcached/create3.png)
![create4.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9B%91%E6%8E%A7memcached/create4.png)
![create5.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9B%91%E6%8E%A7memcached/create5.png)
![create6.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9B%91%E6%8E%A7memcached/create6.png)
![create7.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9B%91%E6%8E%A7memcached/create7.png)
![create8.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9B%91%E6%8E%A7memcached/create8.png)

### 5.2创建触发器

![1563951747536](D:\学习资料\markdow\Basic\zabbix\zabbixsy.assets\1563951747536.png)

### 5.3将模板关联到主机

![add1.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9B%91%E6%8E%A7memcached/add1.png)
![add2.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9B%91%E6%8E%A7memcached/add2.png)
![add3.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9B%91%E6%8E%A7memcached/add3.png)



## 6.监控redis

### 1.安装配置 redis

```bash
#安装
apt-get install redis

#配置redis
vim /etc/redis/redis.conf
...
bind 0.0.0.0
...

#启动服务
systemctl start redis
ss -ntl
6379
```

### 2.测试redis取值

```bash
echo -e "info\n quit" |nc 127.0.0.1 "6379"
```

### 2.配置zabbix-agent，检测redis

```bash
# 编写监控脚本
#!/bin/bash
redis_status(){
        R_PORT=$1
        R_COMMAND=$2
        (echo -en "INFO \r\n";sleep 1;) | ncat 0.0.0.0 "$R_PORT" > /tmp/redis_"$R_PORT".tmp
        REDIS_STAT_VALUE=$(grep ""$R_COMMAND":" /tmp/redis_"$R_PORT".tmp | cut -d ':' -f2)
        echo $REDIS_STAT_VALUE  
}

help(){
        echo "${0} + redis_status + PORT + COMMAND"
}

main(){
    case $1 in
        redis_status)
            redis_status $2 $3
                ;;
        *)
            help
                ;;
        esac
}
main $1 $2 $3

#测试脚本
bash /etc/zabbix/zabbix_agentd.d/redis.sh redis_status 6379 used_cpu_sys
1.04

#给脚本添加执行权限
chmod +x /etc/zabbix/zabbix_agentd.d/redis.sh

#删除测试时候产生的文件夹
rm -rf /tmp/redis_6379.tmp


#添加agent子配置文件
vim /etc/zabbix/zabbix_agentd.d/redis_status.conf
UserParameter=redis_status[*],/etc/zabbix/zabbix_agentd.d/redis.sh $1 $2 $3


#重启服务
systemctl restart zabbix-agent.service 
```

### 3.在zabbix-server上测试能否取值

```bash
zabbix_get  -s 192.168.1.154 -p 10050 -k "redis_status[redis_status 6379 used_cpu_sys]"
0.66
```

### 4.1在server上添加模板

![redis1.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9B%91%E6%8E%A7redis/redis1.png)
![redis2.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9B%91%E6%8E%A7redis/redis2.png)
![redis3.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9B%91%E6%8E%A7redis/redis3.png)
![redis4.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9B%91%E6%8E%A7redis/redis4.png)
![redis5.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9B%91%E6%8E%A7redis/redis5.png)
![redis6.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9B%91%E6%8E%A7redis/redis6.png)

### 4.2将模板关联到主机

![redis7.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9B%91%E6%8E%A7redis/redis7.png)
![redis8.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9B%91%E6%8E%A7redis/redis8.png)
![redis9.png](C:/Users/L/Desktop/cutter/zabbix/zabbix%E7%9B%91%E6%8E%A7redis/redis9.png)

###  5.查看

![1563965782124](D:\学习资料\markdow\Basic\zabbix\zabbixsy.assets\1563965782124.png)





4.7：SNMP
4.7.1：SNMP协议介绍

- SNMP是英文“Simple Network Management Protocol”的缩写，中文意思是”简单网络管理协议“，SNMP是一种简单网络管理协议，它属于TCP/IP五层协议中的应用层协议，用于网络管理的协议，SNMP主要用于网络设备的管理
- SNMP的基本思想：为不同种类的设备、不同厂家生产的设备、不同型号的设备，定义为一个统一的接口和协议，使得管理员可以是使用统一的外观面对这些需要管理的网络设备进行管理。通过网络，管理员可以管理位于不同物理空间的设备，从而大大提高网络管理的效率，简化网络管理员的工作
- SNMP协议是TCP/IP协议簇的一个应用层协议，在1988年被制定，并被Internet体系结构委员会（IAB）采纳作为一个短期的网络管理解决方案，SNMP的协议版本目前有SNMP v1、SNMP v2c和SNMP v3三种版本，其具体差别如下：
  - SNMP v1采用团体名（Community Name）认证，团体名用来定义SNMP NMS和SNMP Agent的关系，如果SNMP报文携带的团体名没有得到设备的认可，该报文将被丢弃，团体名起到了类似于密码的作用，来限制SNMP NMS对SNMP Agent的访问
  - SNMP v2c也采用团体名认证，它在兼容SNMP v1的同时又扩充了SNMP v1的功能，它提供了更多的操作类型（GetBulk和InformRequest）、支持更多的数据类型（Counter64等）、提供了更丰富的错误代码且能够更细致地区分错误
  - SNMP v3提供了基于用户的安全模型（USM，User-Based Security Model）的认证机制，用户可以设置认证和加密功能，认证用于验证报文发送方的合法性，避免非法用户的访问，加密则是对NMS和Agent之间的传输报文进行加密，以免被窃听。通过有无认证和有无加密等功能组合，可以为SNMP NMS和SNMPAgent之间的通信提供更高的安全性  

4.7.2：SNMP工作机制

- SNMP的工作机制通过SNMP网络元素分为NMS和Agent两种：

  - NMS（网络管理站）：是运行SNMP客户端程序的工作站，能够提供非常友好的人机交互界面，方便网络管理员完成绝大多数的网络管理工作

  - Agent是驻留在设备上的一个进程，负责接收、处理来自NMS的请求报文。在一些紧急情况下，如接口状态发生改变等，Agent也会主动通知NMS

  - NMS是SNMP网络的管理者，Agent是SNMP网络的被管理者。NMS和Agent之间通过SNMP协议来交互管理信息‘

4.7.3：SNMP数据交互

- SNMP管理进程与代理进程之间为了交互信息，定义了5种报文：
  - get - request操作：从代理进程处提取一个或多个参数值
  
  - get-next-request操作：从代理进程处提取一个或多个参数的下一个参数值
  
  - set-request操作：设置代理进程的一个或多个参数值

- get-rsponse操作：返回的一个或多个参数值。这个操作是由代理进程发出的

- trap操作：代理进程主动发出的报文，通知管理进程有某些事情发生 

  ![1563966462370](D:\学习资料\markdow\Basic\zabbix\zabbixsy.assets\1563966462370.png)

    4.7.4：SNMP组织结构
  
    SNMP报文协议
    管理信息机构SMI，一套公用的结构和表示符号
    管理信息库MIB，管理信息库包含所有代理进程的所有可被查询和修改的参数
    OID，一个OID是一个唯一的键值对，用于标识具体某一个设备的某个具体信息(对象标识)，如端口信息、设备
    名称等





4.7.4：SNMP组织结构

- 一套完整的SNMP系统主要包括以下几个方面：
  -  SNMP报文协议。
  - 管理信息结构（SMI， Structure ofManagement Information），一套公用的结构和表示符号。
  -  管理信息库（MIB，Management Information Base），管理信息库包含所有代理进程的所有可被查询和修改的参数。
  - OID（Object Identifiers），一个OID是一个唯一的键值对，用于标识具体某一个设备的某个具体信息(对象标识)，如端口信息、设备名称等。

5.4.7.4.1：MIB

- 所谓(MIB)管理信息库，就是所有代理进程包含的、并且能够被管理进程进行查询和设置的信息的集合。MIB是基于对象标识树的，对象标识是一个整数序列，中间以"."分割，这些整数构成一个树型结构，类似于 D N S或U n i x的文            件系统,M I B被划分为若干个组，如s y s t e m、 i n t e r f a c e s、 a t（地址转换）和i p组等。i s o . o r g . d o d . i            n t e r ne t .p r i v a t e . e n t e r p r i s e s（ 1 . 3 . 6 . 1 . 4 . 1）这个标识，是给厂家自定义而预留的，比如华为的            为1.3.6.1.4.1.2011，华三的为1.3.6.1.4.1.25506

  ![1563966653365](D:\学习资料\markdow\Basic\zabbix\zabbixsy.assets\1563966653365.png)

  

  

  

  

4.7.4.2：OID

- Centos部分常用OID

![1563966730237](D:\学习资料\markdow\Basic\zabbix\zabbixsy.assets\1563966730237.png)

- 如何测试OID

  - snmpwalk是SNMP的一个工具，它使用SNMP的GET请求查询指定OID（SNMP协议中的对象标识）入口的所有OID树信息，并显示给用户。通过snmpwalk也可以查看支持SNMP协议（可网管）的设备的一些其他信息，比如cisco交换机或路由器IP地址、内存使用率等，也可用来协助开发SNMP功能。              
  - 要使用snmpwalk需要先安装net-snmp软件包

  ```bash
   yum -y install net-snmp-utils
   [root@s1 ~]# snmpwalk -h
   USAGE: snmpwalk [OPTIONS] AGENT [OID]
   –h：显示帮助。
   –v：指定snmp的版本, 1或者2c或者3。
   –c：指定连接设备SNMP密码。
   –V：显示当前snmpwalk命令行版本。
   –r：指定重试次数，默认为0次。
   –t：指定每次请求的等待超时时间，单为秒，默认为3秒。
   –l：指定安全级别：noAuthNoPriv|authNoPriv|authPriv。
   –a：验证协议：MD5|SHA。只有-l指定为authNoPriv或authPriv时才需要。
   –A：验证字符串。只有-l指定为authNoPriv或authPriv时才需要。
   –x：加密协议：DES。只有-l指定为authPriv时才需要。
   –X：加密字符串。只有-l指定为authPriv时才需要
  ```

- Centos服务器安装配置SNMP：

  ```bash
  root@s6 ~]# yum install -y net-snmp
  [root@s6 ~]# vim /etc/snmp/snmpd.conf
  # sec.name source community
  com2sec notConfigUser default 123456 #第一步：设置团体认证
  group notConfigGroup v1 notConfigUser
  group notConfigGroup v2c notConfigUser #第二步：将团体名称notConfigUser 关联至组notConfigGroup
  view systemview included .1.3.6.1.2.1.1
  view systemview included .1.3.6.1.2.1.25.1.1 #创建一个view，并对其授权可访问的OID范围
  access notConfigGroup “” any noauth exact systemview none none #将组notConfigGroup关联至systemview 从未完成组的授权
  测试能否通过SNMOP采集数据：
  [root@s6 ~]# systemctl restart snmpd
  [root@s1 ~]# snmpwalk -v 2c -c 123456 172.18.200.106 .1.3.6.1.4.1.2021.10.1.3.1
  UCD-SNMP-MIB::laLoad.1 = STRING: 0.00
  [root@s1 ~]# snmpwalk -v 2c -c 123456 172.18.200.106 .1.3.6.1.4.1.2021.4.3.0
  UCD-SNMP-MIB::memTotalSwap.0 = INTEGER: 1048572 kB
  ```

  

## 7.监控mysql

- 使用Ubuntu配置

### 1.配置数据库主从

```bash
#在主服务器 apt安装    192.168.1.153
apt-get install mariadb-server mariadb-client -y

#修改配置文件     
vim /etc/mysql/mariadb.conf.d/50-server.cnf 
server-id               = 12
log_bin                 = /var/log/mysql/mysql-bin.log

#重启服务
systemctl restart mariadb

#授权账户
mysql
show master logs;
reset masters;
grant replication  slave,replication client on *.* to 'repluser'@'192.168.1.%' identified by 'redhat';

#在主服务器上导出所有的数据   完全备份
mysqldump --all-databases --single_transaction --flush-logs --master-data=2 --lock-tables > backup.sql



#在从服务器安装mariadb   192.168.1.154
apt-get install mariadb-server mariadb-client -y

#修改配置文件
vim /etc/mysql/mariadb.conf.d/50-server.cnf
server-id               = 21
log_bin                 = /var/log/mysql/mysql-bin.log

#重启服务
systemctl restart mariadb

#导入备份数据
mysql
mysql < backup.sql 

#从服务器上将主服务器指向192.168.27.12
help change master
CHANGE MASTER TO
       MASTER_HOST='192.168.1.153',
       MASTER_USER='repluser',
       MASTER_PASSWORD='redhat',
       MASTER_PORT=3306,
       MASTER_LOG_FILE='mysql-bin.000001',
        MASTER_LOG_POS=313;

#起复制线程
start slave;

#查看线程是否正常，确保以下两个线程是正常的  
show slave status\G;
```



### 2.从服务器下载监控软件，并安装

- 官方简介
  - 安装procona：
  - 官方文档及下载地址：
  -  https://www.percona.com/doc/percona-monitoring-plugins/LATEST/zabbix/index.html #插件地址
  -  https://www.percona.com/downloads/ #下载地址
  - https://www.percona.com/doc/percona-monitoring-plugins/LATEST/zabbix/index.html#installation-instructions #安装教程

```bash
#下载软件包percona-zabbix-templates
wget https://www.percona.com/downloads/percona-monitoring-plugins/percona-monitoring-plugins-1.1.8/binary/debian/xenial/x86_64/percona-zabbix-templates_1.1.8-1.xenial_all.deb

#安装模板文件
dpkg -i percona-zabbix-templates_1.1.8-1.xenial_all.deb 

#将percona的配置文件复制到zabbix配置文件目录下
cp /var/lib/zabbix/percona/templates/userparameter_percona_mysql.conf /etc/zabbix/zabbix_agentd.d/

#安装php环境
apt-get install php php-mysql -y


#创建一个.cnf的配置文件，原有一个文件不要动
vim /var/lib/zabbix/percona/scripts/ss_get_mysql_stats.php.cnf
<?php
$mysql_user = 'root';
$mysql_pass = '';


#测试获取数值
/var/lib/zabbix/percona/scripts/get_mysql_stats_wrapper.sh gg

#将tmp下生成的测试文件删除
rm -rf /tmp/localhost-mysql_cacti_stats.txt 



```



### 3.在zabbix-server 上测试
```bash
zabbix_get -s 192.168.1.154 -p 10050 -k "MySQL.Key-read-requests"
0
```



### 4.在zabbix-server上导入模板

![1564143225214](D:\学习资料\markdow\Basic\zabbix\zabbixsy.assets\1564143225214.png)

![1564143235562](D:\学习资料\markdow\Basic\zabbix\zabbixsy.assets\1564143235562.png)

- 将模板模式改为主动模式

![1564143258371](D:\学习资料\markdow\Basic\zabbix\zabbixsy.assets\1564143258371.png)

![1564143271540](D:\学习资料\markdow\Basic\zabbix\zabbixsy.assets\1564143271540.png)

![1564143281680](D:\学习资料\markdow\Basic\zabbix\zabbixsy.assets\1564143281680.png)

- 将模板关联到主机

![1564143299659](D:\学习资料\markdow\Basic\zabbix\zabbixsy.assets\1564143299659.png)

![1564143320225](D:\学习资料\markdow\Basic\zabbix\zabbixsy.assets\1564143320225.png)

![1564143331426](D:\学习资料\markdow\Basic\zabbix\zabbixsy.assets\1564143331426.png)

## 7.1自定义监控模板

- 注意：主从同步的数据库三个参数
  - Slave_IO_Running         IO线程
  - Slave_SQL_Running     SQL线程
  - Seconds_Behind_Master   衡量MySQL主备延迟时间

- 推

### 1.在从服务器上编写脚本

```bash
#!/bin/bash
Seconds_Behind_Master(){
    NUM=`mysql -uroot -hlocalhost   -e "show slave status\G;"  | grep "Seconds_Behind_Master:" | awk -F: '{print $2}'`
    echo $NUM
}

master_slave_check(){
NUM1=`mysql -uroot -hlocalhost   -e "show slave status\G;"  | grep "Slave_IO_Running" | awk -F:  '{print $2}' | sed 's/^[ \t]*//g'`
#echo $NUM1
NUM2=`mysql -uroot -hlocalhost   -e "show slave status\G;"  | grep "Slave_SQL_Running:" | awk -F:  '{print $2}' | sed 's/^[ \t]*//g'`
#echo $NUM2
if test $NUM1 == "Yes" &&  test $NUM2 == "Yes";then
    echo 50
else
    echo 100
fi
}

main(){
    case $1 in
    Seconds_Behind_Master)
        Seconds_Behind_Master;
        ;;
    master_slave_check)
        master_slave_check
       ;;
    esac
}
main $1

#给脚本添加执行权限
chmod +x /etc/zabbix/zabbix_agentd.d/mysql_monitor.sh 
```

### 2.添加agent子配置文件，配置key

```bash
#配置子配置文件
UserParameter=mysql_monitor[*],/etc/zabbix/zabbix_agentd.d/mysql_monitor.sh $1

#重启服务
systemctl restart zabbix-agent.service 
```

### 3.在zabbix-server端测试

```bash
zabbix_get -s 192.168.1.21 -p 10050 -k "mysql_monitor[Seconds_Behind_Master]"

```

### 4.使用Ubuntu做遇到的问题

```bash
1. 在zabbix server端执行说 “root@localhost” 没有权限
解决办法：
	方法一，添加一个账户给权限，然后在脚本里面修改账户、密码、和登录IP（127.0.0.1）。
		grant all on *.* to root@'%' identified by '123456';
	方法二,在MySQL配置文件里加上 skip-grant-tables，不需要密码登录。
	方法三，修改root的plugin。
		select user, plugin from mysql.user;
		update mysql.user set plugin='' where user='root';

2. 在远程执行获取数据的时候，报错 mysql_monitor.sh: 12: test: Yes: unexpected operator
原因：Ubuntu的默认sh连接是dash,与bash语法有差别
解决办法：
	方法一：将默认dash修改成bash
	sudo dpkg-reconfigure dash

	方法二：
	将脚本改成dash的，bash中 == 在bash中表示 = 
	所以将脚本修改一下

```

### 5.在zabbix-server上创建相应的模板及监测项

![1564145029902](D:\学习资料\markdow\Basic\zabbix\zabbixsy.assets\1564145029902.png)

![1564145038137](D:\学习资料\markdow\Basic\zabbix\zabbixsy.assets\1564145038137.png)



![1564145047384](D:\学习资料\markdow\Basic\zabbix\zabbixsy.assets\1564145047384.png)

![1564145055249](D:\学习资料\markdow\Basic\zabbix\zabbixsy.assets\1564145055249.png)

![1564145077014](D:\学习资料\markdow\Basic\zabbix\zabbixsy.assets\1564145077014.png)

![1564145085323](D:\学习资料\markdow\Basic\zabbix\zabbixsy.assets\1564145085323.png)

![1564145092881](D:\学习资料\markdow\Basic\zabbix\zabbixsy.assets\1564145092881.png)





### 按照以上流程将master_slave_check的监控项添加到模板内，然后将模板与监控的主机关联。











































































































































































































































































































































































































