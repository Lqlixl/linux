# Zabbix

## 简介

```bash
SNMP 简单网络管理协议 simple Network Management Protocol
专业接口

SNMP协议版本：
v1,v2,v3
v1没有验证工具
v2c 明文的 先定义密钥在agent里面，
v3 支持加密解密认证功能

三种模式：
	NMS向agent采集数据
	agent向NMS报告数据
	NMS请求agent修改配置

SNMP的组件（三个）
	MIB： management information base：每个agent都有自己的MIB库,mib监控对象的集合，名称，权限，是整型，浮点型等等，agent和nms的接口，SNMP太简单所以有规范性文件
	SMI： MIB的表示符号
	SNMP协议本身
NMS可发起操作：
	Get，Getnext(获取多个参数),Set（修改配置）,Trap（陷阱）
agent:Response
	UDP:
	nms:监听在161 端口
	agent:监听在162端口

Linux：使用这个协议需要安装 net-snmp程序包
window自带的


cacti(仙人掌)：从数据库提取数据，然后即时绘制，能报警但是报警能力很弱，本身写一些脚本，发起数据请求，将数据提取到数据库临时存储，php研发的，可以前端显示可以设置时间没五分钟提取一次。



报警：连续检测出现几次软状态，出现硬状态才出现报警。

Nagios工具：调用各种接口，强大的报警工具；主要目的检测从软状态转化成硬状态。即使显示报警。报警升级操作。定义那些时间维护不报警；还支持主键依赖，一个连接一个，第一个出现问题，后面的依赖第一个的接口就不需要报警，只报第一个；eg:路由器出问题，路由器后面接口的服务器等不需要报警；
不关心数据是多少,只管显示


监控系统出问题解决办法：部署多个监控系统，两套独立的运作，也可以两套监控不同的组建；

Ngios+cacti:监控工具 以前使用的

zabbix：

分布式监控

数据存储:
	cacti：rrd(round robin database)自带的数据库，环状的数据存储。每次采集数据是固定。固定的数据 
	zabbix:mysql,pgsql  调用数据库数据库数据

著名的开源监控工具：zabbix,zennos,opennms,cacti,nagios(icinga),ganglia

 监控功能的实现：
 	agent
 	ssh
 	SNMP
 	IPMI

 zabbix:有专业的agent监控工具
 	监控主机：
 		linux、Windos、UNIX(FREEBSD等)
 		监控设备：（网络设备，路由交换机设备）
 		SNMP，SSH（并不是所有）


专用agent:
监控端和被监控端都是Linux系统，直接连接对方输入账户密码。提取数据。
太麻烦，每次提取查看数据都需要数据账户密码，就重新根据自己需要定制agent。
最大问题，很多机器比如路由器不能让装自己定制的agent；


自动化监控
1.数据采集（主动）：通过agent，脚本，snmp协议等
  服务状态检测（被动）：服务状态，程序状态
  第三方信息：公司内相同系统

2.数据处理
复杂计算     阈值判别     有些场景还需要智能分析

3.报警与联动
报警策略       联动处理（报警前先自身处理）    报警跟踪      问题管理

监控评估 
 通过API接口进行二次开发监控功能


可监控对象：
	设备/软件
		设备：服务器、路由器、交换机、IO系统
		软件：OS、网络、应用程序
	偶发性小故障（Incidents）
		主机down机、服务不可以、主机不可达、
	严重性故障（Critical Events）
	主机性能指标
	趋势：时间序列数据（Treads）



触发器触发后启动一个active，active就会选择:报警或者执行脚本。

数据的可视化

```



## A.  apt安装zabbix

- 官网：<https://www.zabbix.com/cn/download?zabbix=4.0&os_distribution=ubuntu&os_version=18.04_bionic&db=mysql>

### Ubuntu安装zabbix

### 1.安装 数据库

```bash
# wget https://repo.zabbix.com/zabbix/4.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_4.0-2+bionic_all.deb
# dpkg -i zabbix-release_4.0-2+bionic_all.deb

设置阿里源
# vim /etc/apt/source.list
更新
# apt update
```

###  2. 安装Zabbix server，Web前端，agent

```bash
# apt -y install zabbix-server-mysql zabbix-frontend-php zabbix-agent
```

### 3. 创建初始数据库

```bash
# mysql 
MariaDB [(none)]> create database zabbix character set utf8 collate utf8_bin;
MariaDB [(none)]> grant all privileges on zabbix.* to zabbix@'192.168.1.%' identified by '123456';
mysql> quit;

#修改MySQL的监听端口为0.0.0.0
ss -ntl

grep "127.0.0.1" /etc/mysql/* -r

vim /etc/mysql/mariadb.conf.d/50-server.cnf
...
bind-address            = 0.0.0.0
...
```

### 4.导入初始架构和数据，系统将提示您输入新创建的密码。

```bash
# zcat /usr/share/doc/zabbix-server-mysql*/create.sql.gz | mysql -uzabbix -p
-h192.168.1.151 zabbix

```

### 5. 为Zabbix server配置数据库

```bash
#修改zabbix的配置文件
root@ubuntu-1804:~# grep "^[a-Z]" /etc/zabbix/zabbix_server.conf 
LogFile=/var/log/zabbix/zabbix_server.log
LogFileSize=0
PidFile=/var/run/zabbix/zabbix_server.pid
SocketDir=/var/run/zabbix
DBHost= 192.168.1.150
DBName=zabbix
DBUser=zabbix
DBPassword= 123456
DBPort=3306
SNMPTrapperFile=/var/log/snmptrap/snmptrap.log
Timeout=4
AlertScriptsPath=/usr/lib/zabbix/alertscripts
ExternalScripts=/usr/lib/zabbix/externalscripts
FpingLocation=/usr/bin/fping
Fping6Location=/usr/bin/fping6
LogSlowQueries=3000
```

### 6. 为Zabbix前端配置PHP

- 编辑配置文件 /etc/zabbix/apache.conf, uncomment and set the right timezone for you.

```bash
#设置有5和7两个版本，查看自己的版本

vim /etc/zabbix/apache.conf
...
php_value date.timezone Asia/Shanghai
...
```

### 7.  启动Zabbix server和agent进程

- 启动Zabbix server和agent进程，并为它们设置开机自启：

```bash
# systemctl restart zabbix-server zabbix-agent apache2
# systemctl enable zabbix-server zabbix-agent apache2
```



## B. 编译安装zabbix 4.0.10
### 1.ubuntu编译安装zabbix 

#### 1.安装mariadb 数据库

```bash
# wget https://repo.zabbix.com/zabbix/4.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_4.0-2+bionic_all.deb
# dpkg -i zabbix-release_4.0-2+bionic_all.deb
# apt update

```



### 2.安装依赖包

```bash
# apt-get  install   apache2 apache2-bin apache2-data apache2-utils fontconfig-config fonts-dejavu-core fping libapache2-mod-php   libapache2-mod-php7.2 libapr1 libaprutil1 libaprutil1-dbd-sqlite3 libaprutil1-ldap libfontconfig1 libgd3 libiksemel3   libjbig0 libjpeg-turbo8 libjpeg8 liblua5.2-0 libodbc1 libopenipmi0 libsensors4 libsnmp-base libsnmp30 libsodium23 libssh2-1  libtiff5 libwebp6 libxpm4 php-bcmath php-common php-gd php-ldap php-mbstring php-mysql php-xml php7.2-bcmath php7.2-cli  php7.2-common php7.2-gd php7.2-json php7.2-ldap php7.2-mbstring php7.2-mysql php7.2-opcache php7.2-readline   php7.2-xml snmpd  ssl-cert ttf-dejavu-core      libmysqlclient-dev  libxml2-dev  libxml2 snmp  libsnmp-dev   libevent-dev  openjdk-8-jdk curl libcurl4-openssl-dev 
```

### 3. 下载编译及安装zabbix 4.0.10

```bash
#cd /opt/

# wget https://sourceforge.net/projects/zabbix/files/ZABBIX%20Latest%20Stable/4.0.10/zabbix-4.0.10.tar.gz/download

# tar xf zabbix-4.0.10.tar.gz

#  cd zabbix-4.0.10/

# ./configure --prefix=/apps/zabbix_server --enable-server --enable-agent --with-mysql --with-net-snmp --with-libcurl --with-libxml2 --enable-java

# make

# make install
```

### 4.编辑zabbix_server.conf配置文件

```bash
#创建用户zabbix 
useradd zabbix 

# 创建日志文件，并授予权限 
mkdir /var/log/zabbix && chown zabbix.zabbix /var/log/zabbix –R
# chown zabbix.zabbix /apps/zabbix_server -R

# grep "^[a-Z]" /apps/zabbix_server/etc/zabbix_server.conf
LogFile=/var/log/zabbix/zabbix_server.log
DBHost= 192.168.1.150
DBName=zabbix
DBUser=zabbix
DBPassword=123456
DBPort= 3306
Timeout=4
LogSlowQueries=3000



```

###  5. 准备数据库

```bash
 # mysql
 # create database zabbix character set utf8 collate utf8_bin;
 
 # grant all privileges on zabbix.* to zabbix@"192.168.1.%" identified by '123456';
 
 #查看
 select host,user,password from mysql.user;

#查看监听地址为本地的 127.0.0.1
grep "127.0.0.1" /etc/mysql/* -r
vim /etc/mysql/mariadb.conf.d/50-server.cnf
...
bind-address            = 0.0.0.0
...

#测试 192.169.1.% 的zabbix用户 可不可以登录
mysql -uzabbix -p123456 -h192.168.1.150
```

### 6. 数据库初始化：

```bash
#导入数据库
cd /opt/zabbix-4.0.10/database/mysql
ls
data.sql  images.sql  Makefile  Makefile.am  Makefile.in  schema.sql

# 导入文件
mysql -uzabbix -p123456 -h192.168.1.150 zabbix < schema.sql   #zabbix是数据库名 
mysql -uzabbix -p123456 -h192.168.1.150 zabbix < images.sql 
mysql -uzabbix -p123456 -h192.168.1.150 zabbix < data.sql 

#临时启动服务查看
/apps/zabbix_server/sbin/zabbix_server -c /apps/zabbix_server/etc/zabbix_server.conf
#查看端口
ss -ntl
10051 是zabbox的服务端口
```

### 5.启动apache2：

> systemctl start apache

### 6.访问页面

```bash
#yum install httpd -y
# systemctl restart httpd 访问web

#删除原来的界面
rm -rf /var/www/html/*
#复制zabbix的web界面
cp -a /opt/zabbix-4.0.10/frontends/php/* /var/www/html/

#访问
http://192.168.1.150/setup.php
```

### 7. 解决报错

#### 7.1 解决php依赖

```bash
# apt-get install php-gettext php-xml php-net-socket php-gd php-mysql

#查找位置
find / -name php.ini
/etc/php/7.2/cli/php.ini
/etc/php/7.2/apache2/php.ini

#修改配置文件
vim /etc/php/7.2/apache2/php.ini
post_max_size = 8M 改为 post_max_size = 16M
max_execution_time = 30 改为 max_execution_time = 300
max_input_time = 60 改为 max_input_time = 300
;date.timezone = 改为 date.timezone = date.timezone = Asia/Shanghai

#将zabbix的配置文件修改
cd /var/www/html/
cp zabbix.conf.php.example zabbix.conf.php
grep "^[a-Z\$]" zabbix.conf.php
global $DB, $HISTORY;
$DB['TYPE']				= 'MYSQL';
$DB['SERVER']			= '192.168.1.150';
$DB['PORT']				= '3306';
$DB['DATABASE']			= 'zabbix';
$DB['USER']				= 'zabbix';
$DB['PASSWORD']			= '123456';
$DB['SCHEMA']			= '';
$ZBX_SERVER				= '192.168.1.150';
$ZBX_SERVER_PORT		= '10051';
$ZBX_SERVER_NAME		= 'zabbix';
$IMAGE_FORMAT_DEFAULT	= IMAGE_FORMAT_PNG;


#重启apache2的服务
systemctl restart apache2.service

# 在此进入网页报错解决了
http://192.168.1.150/setup.php
```

#### 7.2  数据库信息

```bash
#填入信息是这个配置文件里的
grep "^[a-Z]" /apps/zabbix_server/etc/zabbix_server.conf

DBHost= 192.168.1.150
DBName=zabbix
DBUser=zabbix
DBPassword=123456
DBPort= 3306
```

#### 7.3 zabbix信息

```bash
host  192.168.1.150
port 10051
Name zabbix
```

### 8. 界面

```bash
http://192.168.1.150/zabbix.php?action=dashboard.view
```

### 9. 界面显示中文

```bash
英文Ubuntu系统安装中文支持，中文UTF-8
#第一步：安装中文包
apt-get install language-pack-zh*

#第二步，配置相关环境变量
vim /etc/environment 
在文件中添加语言和编码设置：
LANG="zh_CN.UTF-8"
LANGUAGE="zh_CN:zh:en_US:en"

配置文件显示：
root@ubuntu-1804:~# cat /etc/environment
PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games"
LANG="zh_CN.UTF-8"
LANGUAGE="zh_CN:zh:en_US:en"
UNZIP="-O CP936"
ZIPINFO="-O CP936"

#第三步，重新设置本地配置：
dpkg-reconfigure locales

```

### 10  解决显示中文出现的乱码

```bash
#检测图形,CPU load出现乱码
find / -name assets
#存放文字类型的位置
ll /var/www/html/assets/fonts/
#在windows 搜索字体，将字体传入文件/var/www/html/assets/fonts/下

#修改配置文件
find / -name defines.inc.php
/var/www/html/include/defines.inc.php
/opt/zabbix-4.0.10/frontends/php/include/defines.inc.php
#将DejaVuSans字体改为simkai字体
vim /var/www/html/include/defines.inc.php
...
define('ZBX_FONT_NAME', 'simkai');
...
define('ZBX_GRAPH_FONT_NAME',           'simkai');
...
graphfont.ttf
```

### 11. 启动脚本编写

- 临时启动服务。必须使用kill 

```bash

#创建pid 启动文件存放位置并且授权
mkdir /apps/zabbix_server/run
chown zabbix.zabbix /apps/zabbix_server/run/

ll /lib/systemd/system/zabbix-server.service
cat /lib/systemd/system/zabbix-server.service
[Unit]
Description=Zabbix Server
After=syslog.target
After=network.target

[Service]
Environment="CONFFILE=/apps/zabbix_server/etc/zabbix_server.conf"
EnvironmentFile=-/etc/default/zabbix-server
Type=forking
Restart=on-failure
PIDFile=/apps/zabbix_server/run/zabbix_server.pid
KillMode=control-group
ExecStart=/apps/zabbix_server/sbin/zabbix_server -c $CONFFILE
ExecStop=/bin/kill -SIGTERM $MAINPID
RestartSec=10s
TimeoutSec=infinity

[Install]
WantedBy=multi-user.target



```

## CentOS安装

### 1.安装包

```bash
#最小化系统需要安装的包
yum install vim iotop bc gcc gcc-c++ glibc glibc-devel pcre pcre-devel openssl openssl-devel zip unzip zlib-devel net-tools lrzsz tree ntpdate telnet lsof tcpdump wget libevent libevent-devel -y

#编译需要安装的包
yum install gcc libxml2-devel net-snmp net-snmp-devel curl curl-devel php php-bcmath php-mbstring mariadb mariadb-devel -y

```

### 2.创建数据库

```bash
#zabbix数据库
create database zabbix character set utf8 collate utf8_bin;
#授权账户
grant all privileges on zabbix.* to zabbix@"192.168.1.%" identified by '123456';

```

### 3.编译安装

```bash
 #创建zabbix用户
 useradd zabbix -s /sbin/nologin
 
 cd /usr/local/zabbix-4.2.4
 
 ln -sv zabbix-4.2.4/ zabbix
 #编译
 ./configure --enable-server --enable-agent --with-mysql --enable-ipv6 --with-net-snmp --with-libcurl --with-libxml2
 
 make && make install

```

### 4.导入数据库，初始化数据库

```bash
cd /usr/local/zabbix-4.2.4/database/mysql/
#导入数据库
mysql -uzabbix -p123456 -h192.168.1.27 zabbix < schema.sql
mysql -uzabbix -p123456 -h192.168.1.27 zabbix < images.sql
mysql -uzabbix -p123456 -h192.168.1.27 zabbix < data.sql
```

### 6.启动脚本

```bash
#查看启动脚本
ll /usr/local/zabbix-4.2.4/misc/init.d/fedora/core/zabbix_server

cp /usr/local/zabbix-4.0.1/misc/init.d/fedora/core/zabbix_server /etc/init.d/
cp /usr/local/zabbix-4.0.1/misc/init.d/fedora/core/zabbix_agentd /etc/init.d/

#修改启动脚本配置参数
21 # Zabbix-Directory
22 BASEDIR=/usr/local/zabbix
```

### 7.编辑zabbix的配置文件

```bash
#创建存放pid log 的配置文件
mkdir /var/log/zabbix && chown zabbix.zabbix /var/log/zabbix –R

#查看配置文件
grep "^[a-Z]" /usr/local/zabbix-4.2.4/conf/zabbix_server.conf
LogFile=/usr/local/zabbix/zabbix_server.log
PidFile=/usr/local/zabbix/zabbix_server.pid
DBHost=192.168.1.27
DBName=zabbix
DBUser=zabbix
DBPassword=123456
Timeout=30
LogSlowQueries=3000
StatsAllowedIP=127.0.0.1


```





































































































































































































































































































































