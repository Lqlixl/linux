1.总结haproxy的调度算法。

2.基于cookie实现客户端会话保持。

3.实现http模式下的客户端IP透传/后端服务器的状态监测机制等配置选项。

4.总结4层负载和七层代理的区别。

##  HAProxy

#### 1.安装HAProxy  

1. yum安装

\```bash

yum install haproxy.x86_64 -y

*#查看版本*

 haproxy -v

HA-Proxy version 1.5.18 2016/05/10

Copyright 2000-2016 Willy Tarreau <willy@haproxy.org>

*#查看服务*

rpm -ql haproxy

3启动服务

systemctl start haproxy.service

*#查看启动脚本，1.5版本与新版本的不一样，有差异。*

cat /usr/lib/systemd/system/haproxy.service 

[Unit]

Description=HAProxy Load Balancer

After=syslog.target network.target

[Service]

EnvironmentFile=/etc/sysconfig/haproxy

ExecStart=/usr/sbin/haproxy-systemd-wrapper -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid $OPTIONS

ExecReload=/bin/kill -USR2 $MAINPID

KillMode=mixed

[Install]

WantedBy=multi-user.target

*# 查看进程，是串联的进程号。*

ps -ef |grep haproxy

root      12817      1  0 09:42 ?        00:00:00 /usr/sbin/haproxy-systemd-wrapper -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid

haproxy   12818  12817  0 09:42 ?        00:00:00 /usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -Ds

haproxy   12819  12818  0 09:42 ?        00:00:06 /usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -Ds

\```

2. 编译安装  

\```bash

yum install gcc gcc-c++ glibc glibc-devel pcre pcre-devel openssl openssl-devel systemd-devel net-tools vim iotop bc zip unzip zlib-devel lrzsz tree screen lsof tcpdump wget ntpdate -y

*#官网下载安装包 www.haproxy.org*

cd /usr/local/src/

tar xvf haproxy-1.8.16.tar.gz && cd haproxy-1.8.20

*#查看怎样安装*

vim /usr/local/src/README

*#安装*

make ARCH=x86_64 TARGET=linux2628 USE_PCRE=1 USE_OPENSSL=1 USE_ZLIB=1 USE_SYSTEMD=1 USE_CPU_AFFINITY=1 PREFIX=/usr/local/haproxy

make install PREFIX=/usr/local/haproxy

*#查看版本*

/usr/local/src/haproxy-1.8.20/haproxy -v

*#查看参数*

/usr/local/src/haproxy-1.8.20/haproxy -h

*# 移动启动位置*

cp haproxy /usr/sbin/

*#创建启动脚本*

cat /usr/lib/systemd/system/haproxy.service

[Unit]

 Description=HAProxy Load Balancer

After=syslog.target network.target

[Service]

ExecStartPre=/usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg -c -q

ExecStart=/usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid

ExecReload=/bin/kill -USR2 $MAINPID

[Install]

WantedBy=multi-user.target

*#创建存放pid的目录*

mkdir /usr/local/haproxy/run

会自动生成mkdir /usr/local/haproxy/run/haproxy.pid

*#创建用户和目录*

mkdir /etc/haproxy

useradd haproxy -s /sbin/nologin

mkdir /var/lib/haproxy

chown haproxy.haproxy /var/lib/haproxy/ -R

systemctl start haproxy

*#查看配置文件*

需要添加配置文件

cat /etc/haproxy/haproxy.cfg 

global 下的是全局配置

defaults 默认参数

haproxy.cfg文件中定义了chroot、pidfile、user、group等参数，如果系统没有相应的资源会导致haproxy无法启动，具体参考日志文件/var/log/messages

*#启动HAProxy：*

systemctl enable haproxy && systemctl restart haproxy

\```

3. Ubuntu 安装  

\> apt-get install haproxy

#### 2.HAProxy组成

1. 程序环境：

\- 主程序：/usr/sbin/haproxy

\- 配置文件：/etc/haproxy/haproxy.cfg

\- Unit file：/usr/lib/systemd/system/haproxy.service

2. 配置段：

\- global：全局配置段

进程及安全配置相关的参数  

性能调整相关参数  

Debug参数,日志等级  

\- proxies：代理配置段

defaults：为frontend, backend, listen提供默认配置  

frontend：前端，相当于nginx中的server {}  

backend：后端，相当于nginx中的upstream {}  

listen：同时拥有前端和后端配置  

3. Global配置

• chroot #锁定运行目录  

• deamon #以守护进程运行  

• #stats socket /var/lib/haproxy/haproxy.sock mode 600 level admin #socket文件  

• user, group, uid, gid #运行haproxy的用户身份  

• nbproc #开启的haproxy进程数，与CPU保持一致  

• nbthread #指定每个haproxy进程开启的线程数，默认为每个进程一个线程  

• cpu-map 1 0 #绑定haproxy 进程至指定CPU  

• maxconn #每个haproxy进程的最大并发连接数  

• maxsslconn #SSL每个haproxy进程ssl最大连接数  

• maxconnrate #每个进程每秒最大连接数  

• spread-checks #后端server状态check随机提前或延迟百分比时间，建议2-5(20%-50%)之间  

• pidfile #指定pid文件路径  

• log 127.0.0.1 local3 info #定义全局的syslog服务器；最多可以定义两个  

4. 

5. frontend/ backend 配置案例

\```bash

*#使用 frontend 和 backed配置*

frontend web_port_80

 bind 172.20.76.68:80

 mode http

 use_backend web_host

backend web_host

mode http

 server web1 172.20.76.38:80

 server web2 172.29.76.48:80

\```

6. Proxies配置- listen

listen web 

​    bind 172.20.76.68:80

   server web1 192.168.1.38:80

   server web1 192.168.1.48:80

-  bind：指定HAProxy的监听地址，可以是IPV4或IPV6，可以同时监听多个IP或端口，可同时用于listen字段中 ，写两个地址来  隔开，一个给防火墙使用，一个给内部服务器调用使用  
- mode http/tcp #指定负载协议类型
- mode http/tcp #指定负载协议类型
- option #配置选项
- server #定义后端real server
• check #对指定real进行健康状态检查，默认不开启  
• addr IP #可指定的健康状态监测IP  
• port num #指定的健康状态监测端口  
• inter num #健康状态检查间隔时间，默认2000 ms  
• fall num #后端服务器失效检查次数，默认为3  
• rise num #后端服务器从下线恢复检查次数，默认为2  
• weight #默认为1，最大值为256，0表示不参与负载均衡  
• backup #将后端服务器标记为备份状态  
• disabled #将后端服务器标记为不可用状态  
• redirect prefix http://www.xiangzi.com/ #将请求临时重定向至其它URL，只适用于http模式  
• maxconn <maxconn>：当前后端server的最大并发连接数  
• backlog <backlog>：当server的连接数达到上限后的后援队列长度   
```bash
listen web_port_80
    mode tcp 
    bind 172.20.76.68:80 172.20.76.68：82
   server web1 172.20.76.38:80 weight 1 check port 9000 inter 3s fall 3 rise 5 
   server web2 172.20.76.48:80 weight 1 check port 9000 inter 3s fall 3 rise 5
   server web2 172.20.76.58:80 weight 1 check port 9000 inter 3s fall 3 rise 5 backup
# bind 可以设置两个IP或者两个端口，隔开一个给防火墙使用，一个给内部服务器调用使用。
# 将server后面的服务名字 web1 web2 修改成ip号，以后号辨别机器
# 监测 服务器都挂了，时间间隔3秒每次，检测3次，启动备用服务器，服务器修好，9秒检测成功，备份服务器就在此交接。
设置七层透传 http  option forwardfor
设置四层透传 tcp     send-proxy
server 172.20.76.68 blogs.studylinux.net:80 send-proxy check inter 3000 fall 3 rise 5

```
#### 3.Cookie 配置  

- 基于cookie实现的session保持

- cookie <value>：为当前server指定cookie值，实现基于cookie的会话黏性

- cookie <name> [ rewrite | insert | prefix ] [ indirect ] [ nocache ] [ postonly ] [preserve ] [ httponly ] [ secure ] [ domain <domain> ]* [ maxidle <idle> ] [maxlife <life> ]

  ```bash
  listen web_port_80
      mode http
      bind 172.20.76.68:80 
      option forwardfor
      cookie SERVER-COOKIE insert indirect nocache
     server 172.20.76.38 172.20.76.38:80  cookie web-38 check port 9000 inter 3s fall 3 rise 5       
     server 172.20.76.48 172.20.76.48:80  cookie web-48 check port 9000 inter 3s fall 3 rise 5
     server 172.20.76.58 172.20.76.58:80  cookie web-58 check port 9000 inter 3s fall 3 rise 5 backup
    基于cookie需要修改头部，作用在HTTP上
    
  ```

  

####  4.配置HAProxy状态页

  ```bash
  listen stats
   bind :9527
   stats enable    启动状态页
   stats hide-version  隐藏版本
   stats uri /haproxy-status 定义状态页的uri
   stats realm HAPorxy\ Stats\ Page
   stats auth haadmin:admin   账户
   stats auth admin:redhat    密码
   stats refresh 30s  自动刷新间隔
   stats admin if TRUE   if是设置管理功能
  
  ```

  #### 5.HAProxy 日志配置

- 日志内容很少，需要自己在default配置项定义；

  ```bash
  
  # log 127.0.0.1 local{1-7} info #基于syslog记录日志到指定设备，级别有(err、
  warning、info、debug)
  #配置rsyslog:
  vim /etc/rsyslog.conf
  
  $ModLoad imudp
  
  $UDPServerRun 514
  
  local3.*       /var/log/haproxy_linux.log
  #配置HAProxy
  在配置文件里面的listen里添加
  ll /etc/haproxy/haproxy.cfg
  
  *#重启日志服务和haproxy*
  
  systemctl restart rsyslog.service 
  
  systemctl restart haproxy.service
  
  *#查看日志* 
  
  ll /var/log/haproxy_linux.log 
  或者访问状态页
  ```

  - 一般不会在haproxy上记录大量日志，这里是公司的总代理器，很多东西在上面，记录大量日志，会影响服务器的性能，记录在后端服务器nginx上面。

####  6.Web服务器状态检测

- 三种状态检测方式：

  1.基于四层的传输端口做状态检测

  > server 172.20.76.38 172.20.76.38:80  check port 9000 inter 3s fall 3 rise 5       

  2.基于指定URI 做状态监测,能正常访问后端服务器的这个页面，证明后端服务就是正常的，就在线，不能访问这个页面，就证明服务器有问题，就down掉，不能工作。

  ```bash
  # 在后端服务器172.20.76.48上建立页面
  mkdir /var/www/html/monitor_page
  echo 48 OK > /var/www/html/monitor_page/index.html
  #haproxy上listen下添加信息
  vim /etc/haproxy/haproxy.cfg
  option httpchk GET /monitor_page/index.html  HTTP/1.0
  #重启服务
  systemctl restart httpd.service
  #访问haproxy状态页
  http://172.20.76.68:9527/haproxy-status
  ```
```
  
3.基于指定URI的request请求头部内容做状态监测  
  
- 监控后端服务器页面，只看头部head，是200状态就证明正常。
  
    ```bash
    #在刚刚后端服务器有monitor——page页面基础上
    # 在haproxy配置文件里listen下添加信息
    vim /etc/haproxy/haproxy.cfg
    option httpchk HEAD /monitor_page/index.html HTTP/1.0\r\nHost:\ 172.20.76.68
  # 重启服务
  systemctl restart haproxy.service
    # 查看172.20.76.48登录日志
    tail /var/log/httpd/access_log
    "-   172.20.76.68 - - [07/Jun/2019:21:32:43 +0800] "HEAD/monitor_page/index.htmlHTTP/1.0" 200 - "-" "-"
    # 查看172.20.76.38登录日志
    tail /var/log/httpd/access_log
    "-\  172.20.76.68 - - [07/Jun/2019:21:33:21 +0800] "HEAD/monitor_page/index.htmlHTTP/1.0" 404 - "-" "-"
    # 有monitor_page页面的头部信息为 200
    # 没有monitor_page页面的头部信息为  404
    # 这样来判断后端服务是否正常
```

    ### HAProxy-服务器动态上下线

```bash
#安装工具
yum install socat -y
#在配置文件里添加进程,在这儿有多少进程就添加几行
vim /etc/haproxy/haproxy.cfg
stats socket /var/lib/haproxy/haproxy.sock1 mode 600 level admin process 1
stats socket /var/lib/haproxy/haproxy.sock2 mode 600 level admin process 2
#重启服务
systemctl restart haproxy
#关闭后端服务器172.20.76.38的服务，需要将所有工作进程都关闭。关闭会空一行。
echo "enable server web_port_80/172.20.76.38" |socat stdio /var/lib/haproxy/haproxy.sock1

echo "enable server web_port_80/172.20.76.38" |socat stdio /var/lib/haproxy/haproxy.sock2

#打开后端服务器172.20.76.38的服务，需要打开所有工作进程，打开会空一行。
echo "enable server web_port_80/172.20.76.38" |socat stdio /var/lib/haproxy/haproxy.sock1

echo "enable server web_port_80/172.20.76.38" |socat stdio /var/lib/haproxy/haproxy.sock2

```
### ACL

- ACL在七层模式下才生效，要是在四层下，直接跳走。

- haproxy支持用户对请求报文做处理，在第一阶段就做判断，比如匹配host,基于不同的host分配到不同的后端服务器。host就是访问的域名。

#### 1.acl参数

  - acl：对接收到的报文进行匹配和过滤，基于请求报文头部中的源地址、源端口、目标地
    址、目标端口、请求方法、URL、文件后缀等信息内容进行匹配并执行进一步操作。

  - acl   "aclname" "criterion"    [flags]          [operator]        [value]

  - acl         名称       条件      条件标记位    具体操作符    操作对象类型

  - 实例

    > acl image_service hdr_dom(host) -i img.magedu.com

  - ACL名称，可以使用大字母A-Z、小写字母a-z、数字0-9、冒号：、点.、中横线和下划线，并且严格区分大小写，必须Image_site和image_site完全是两个acl。

```bash
#Criterion-acl:
1.ACL derivatives
• hdr（[<name> [，<occ>]]）：完全匹配字符串
• hdr_beg（[<name> [，<occ>]]）：前缀匹配
• hdr_dir（[<name> [，<occ>]]）：路径匹配
#• hdr_dom（[<name> [，<occ>]]）：域匹配
• hdr_end（[<name> [，<occ>]]）：后缀匹配
• hdr_len（[<name> [，<occ>]]）：长度匹配
• hdr_reg（[<name> [，<occ>]]）：正则表达式匹配
• hdr_sub（[<name> [，<occ>]]）：子串匹配
2.<criterion>:匹配条件
• dst 目标IP
• dst_port 目标PORT
• src 源IP
• src_port 源PORT
#flags
<flags>-条件标记
• -i 不区分大小写
• -m 使用指定的pattern匹配方法
• -n 不做DNS解析
• -u 禁止acl重名，否则多个同名ACL匹配或关系
#operator
1.operator]-操作符：
整数比较：eq、ge、gt、le、lt
2.字符比较：
....
#Value
<value>的类型：
• - Boolean #布尔值
• - integer or integer range #整数或整数范围，比如用于匹配端口范围
• - IP address / network #IP地址或IP范围, 192.168.0.1 ,192.168.0.1/24
• - string 
• - regular expression #正则表达式
• - hex block #16进制
#Acl定义与调用
acl作为条件时的逻辑关系：
• - 与：隐式（默认）使用
• - 或：使用“or” 或 “||”表示
• - 否定：使用“!“ 表示
	示例：
	if valid_src valid_port #与关系
	if invalid_src || invalid_port #或
	if ! invalid_src #非
```

#### 2.域名匹配 

- 基于haproxy ACL实现域名匹配，把不同的域名请求调度到不同的后端服务器

```bash
#设置haproxy配置文件：172.20.76.68
vim /etc/haproxy/haproxy.cfg
frontend web 
    bind :80	#监听本机的80端口 
    mode http	#使用http协议
    #定义acl规则
    acl pc_web_page hdr_dom(host) -i pc.xiangzi.net
     # 给acl取名    使用域名匹配 不区分大小写 域名
    acl mobile_web_page hdr_dom(host) -i mobile.xiangzi.net
    #调用下面backend的定义（像函数调用下方定义的参数）
    use_backend pc_web_host if  pc_web_page
    #       下方的参数符合 上方定义的一个 acl规则
    use_backend mobile_web_host if  mobile_web_page
    #默认服务器，上面偶读不符合就去下面默认的服务器（有的域名）
    default_backend backup_web_host

#定义的后端服务器
backend pc_web_host
 server 172.20.76.38  172.20.76.38:80 check inter 3s fall 3 rise 5

backend mobile_web_host
 server 172.20.76.48 172.20.76.48:80 check inter 3s fall 3 rise 5

backend backup_web_host
 server 172.20.76.58 172.20.76.58:80 check inter 3s fall 3 rise 5
 #重启服务，重新加载
 systemctl reload haproxy
 systemctl restart haproxy
 # 配置后端pc端web服务器172.20.76.38
 安装有http服务
 echo pc_web 38 page > /var/www/html/index.html
 systemctl restart httpd 
 # 配置后端mobile端web服务器172.20.76.48
 echo mobile_web 48 page > /var/www/html/index.html
 systemctl restart httpd 
 # 配置后端默认web服务器172.20.76.58
 echo default_web 58 page > /var/www/html/index.html
 systemctl restart httpd
 
 #自己做实验需要在电脑里配置域名
 C:\Windows\System32\drivers\etc
 cat /etc/hosts
172.20.76.68 www.xiangzi.com www.xiangzi.net pc.xiangzi.net mobile.xiangzi.net
 #测试访问
 curl -L http://pc.xiangzi.net/
  pc_web 38 page
 curl -L http://mobile.xiangzi.net/
  mobile_web 38 page
 curl -L http://www.xiangzi.com
 default_web 58 page
 
```

#### 3.域名重定向

- 将访问的域名重定向到新的域名地址，eg:很多人保存有网页书签，但是修改过域名网址，保存的书签就不生效了,就使用域名重定向解决。

  ```bash
   在haproxy配置文件里添加一行重定向
   vim /etc/haproxy/haproxy.cfg
   frontend web 
      bind :80 
      mode http
    #最开始的域名
   acl pc_web_page hdr_dom(host) -i pc.xiangzi.net
    #重定向到新的域名。www.xiangzi.com
    redirect prefix http://www.xiangzi.com if pc_web_page
    #定义新的域名acl规则
      acl terminal_web_page hdr_dom(host) -i www.xiangzi.com
      #调用下面的后端服务器
   use_backend pc_web_host if  terminal_web_page 
   
   backend pc_web_host
   server 172.20.76.38  172.20.76.38:80 check inter 3s fall 3 rise 5
  
  #重启测试
  curl -L http://pc.xiangzi.net
  web_38
  crul -L http://www.xiangzi.com
  web_38
  ```

#### 4.自定义错误页面跳转

- 设置跳转到本机的错误页面
  
  ```bash
    1.#在haproxy的主配置文件里面设置
    vim /etc/haproxy/haproxy.cfg
    #在default下添加设置
    errorfile 500
    /usr/local/haproxy/html/500.html #自定义错误页面跳转
    errorfile 502 /usr/local/haproxy/html/502.html
    errorfile 503 /usr/local/haproxy/html/503.html
    #创建错误页面（一般找开发设计页面）
    mkdir /usr/local/haproxy/html
    echo 500 fail > /usr/local/haproxy/html/500.html
    echo 501 fail > /usr/local/haproxy/html/501.html
    echo 503 fail > /usr/local/haproxy/html/503.html
    #重启服务
    systemctl restart haproxy
    
    2.#将错误页面跳转到专门报错的服务器上
    vim /etc/haproxy/haproxy.cfg
    #在defaulet设置加添加将错误503页面重定向到其他服务器上
    errorloc 503 http://172.20.76.28/error_page/503.html
    #重启服务
    systemctl restart haproxy
    #在后端用于专门报错的服务器上172.20.76.28
    编译的，存放目录有区别
    mkdir /apps/nginx/html/error_page
    echo "you fail,you should trys again">
    /apps/nginx/html/error_page/index.html
    
    #测试：
    curl -L http://172.20.76.28/error_page/503.html
    you fail,you should trys again
    
  ```
  
#### 5.基于acl+文件后缀实现动静分离

- 将是后缀名 .jpg .png .jpeg .gif的设置访问172.20.76.48，存放图片
- 将后缀名是 .php 的设置访问172.20.76.38，存放数据

```bash
#设置haproxy配置文件 172.20.76.68
vim /ehc/haproxy/haproxy.cfg
frontend web 
    bind :80 
    mode http
    #设置acl规则
    acl php_server path_end -i .php
    acl image_server path_end -i .jpg .png .jpeg .gif
	#调用上方的规则。和下方设置的设置（函数调用一般）
    use_backend php_web_host if php_server
    use_backend image_web_host if image_server
    default_backend default_web_host
        
backend php_web_host
 server 172.20.76.38  172.20.76.38:80 check inter 3s fall 3 rise 5

backend image_web_host
 server 172.20.76.48 172.20.76.48:80 check inter 3s fall 3 rise 5

backend default_web_host
server 172.20.76.58 172.20.76.48:80 check inter 3s fall 3 rise 5
#重置服务器
systemctl restart haproxy

#设置存放数据的服务器.php后缀的，172.20.76.48
yum install naginx php-fpm -y
systemctl start nginx php-fpm -y
# 添加配置文件
vim /etc/nginx/nginx.conf
#在主location下添加
  location ~ \.php$ {
        root           html;
             fastcgi_pass   127.0.0.1:9000;
             fastcgi_index  index.php;
             fastcgi_param  SCRIPT_FILENAME  /$document_root$fastcgi_script_name;   #页面显示主页下，默认脚本路径。
             include        fastcgi_params;
          }   
#添加.php测试页面
vim /usr/share/nginx/html/index.php
	<?php
		phpinfo();                                                             
	?>
nginx -t
systemctl restart nginx.service

#设置存放图片后缀的后端服务器 172.20.76.48
yum isntall httpd -y
ll /var/www/html
	1.jpg  2.png
systemctl start httpd

#测试
172.20.76.68 www.xiangzi.com pc.xiangzi.net mobile.xiangzi.net www.borrow.net mac.xiangzi.net www.xiangzi.net

http://www.xiangzi.com/1.jpg

#图片也可以设置访问路径的方式
#在haproxy的配置文件里面
acl static_path path_beg -i /static /images /javascript
#在存储图片的服务器里将图片放在 /static 或者/images下就行了

```





















