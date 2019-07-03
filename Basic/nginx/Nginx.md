## web服务端  
### 应用程序工作模式  
1. Apache prefork 模型：  
预派生模式，有一个主控制进程，然后生成多个子进程，使用select模型，最大并发1024，每个子进程有一个独立的线程响应用户请求，相对比较占用内存，但是比较稳定，可以设置最大和最小进程数，是最古老的一种模式，也是最稳定的模式，适用于访问量不是很大的场景。  
优点：稳定；不用担心线程安全问题。  
缺点：慢，占用资源，1024个进程不适用于高并发场景；  
2. Apache woker模型：  
一种多进程和多线程混合的模型，有一个控制进程，启动多个子进程，每个子进程里面包含固定的线程，使用线程来处理请求，当线程不够使用的时候在再启动一个新的子进程，然后在进程里面再启动线程处理请求，由于其使用了线程处理请求，因此可以承受更高的并发。  
优点：相比prefork占用的内存较小，可以同时处理更多的请求：  
缺点：使用keepalive的长连接方式，某个线程会一直被占据，即使没有传输数据，也需要一直等待到超时才被释放。如果过多的线程，被这样占据，也会导致在高并发场景下的无服务线程可用。（该问题在prefork模式下，也同样会发生）；  
多线程下多进程的原因，如果一个线程异常挂掉，会导致父进程连同其他正常的子线程都挂掉（在同一个进程下），为了防止这种异常场景出现，就不能全部使用线程，要使用多线程加多进程。  
注：keep-alive 的长连接方式，是为了让下一次的socket通信复用之前创建的连接，从而，减少连接的创建和销毁的系统开销。保持连接，会让某个进程或者线程一直处于等待状态，即使数据没有过来。
3. Apache event模型：  
Apache中最新的模式，2012年发布的apache2.4x系列正式支持event模型，属于事件驱动模型（epoll）,每个进程响应多个请求，在现在版本里的已经是稳定可用的模式。它和worker很像，最大的区别在于，它解决了keepalive场景下，长期被占用的线程的资源浪费问题（某些线程因为被keepalive,空挂在哪儿等待，中间几乎没有请求过来，甚至等到超时）。event MPM中，会有一个专门的线程来管理这些keepalive类型的线程，当有真实请求过来，将请求转递给服务线程，执行完毕后，又允许它释放。这样增强了高并发场景下的请求处理能力；  
监听线程用于向工作线程分配任务并和客服端保持会话连接，超时之后监听线程会删除该socket，工作线程值处理用户请求，处理完之后将会话保持交予监听线程，自己去处理新的请求，不在负责会话保持。  
优点：单线程响应多请求，占据更少的内存，高并发下表现更优秀，会又一个专门的线程来管理keepalive类型的线程，当有真实请求过来的时候，将请求传递给服务线程，执行完毕后，又允许它释放：  
缺点：没有线程安全控制;   
4. Nginx（Master+worker）模式  
主进程  
工作进程 #直接处理客服请求  
线程验证方式：  
>cat /proc/PID/status  
>pstree -p PID  
### 性能  
1. 性能访问体验统计  
互联网存在用户速度体验的1-3-10原则，既1秒优先，1-3秒较优，3-10秒比较慢，10秒以上用户无法接受，用户放弃一个产品代价很低，只是换一个URL而已。  
2. 性能影响：  
性能对用户行为有很大的影响：79%的用户表示不太可能在此打开一个缓慢的网站。  
8秒定律：用户访问一个网站时，如果等待网页打开的时间超过8秒，会有30%的用户放弃等待。  
3. 影响用户体验的几个因素：  
- 客服端硬件配置  
- 客服端网络速率  
- 客服端与服务端距离  
- 服务端网络速率  
- 服务端硬件配置  
- 服务端架构设计  
- 服务端应用程序工作程序  
- 服务端并发数量  
- 服务端响应文件大小及数量  
- 服务端I/O压力  
4. 服务端I/O：  
I/O在计算机中指input/Output，IOPS

























## 编译安装nginx  
1. 安装编译需要的安装包和最小化安装需要安装的一些要使用的包  
> yum install vim lrzsz tree screen psmisc lsof tcpdump wget ntpdate gcc gcc-c++ glibc glibc-devel pcre pcre-devel openssl openssl-devel systemd-devel net-tools iotop bc zip unzip zlib-devel bash-completion nfs-utils automake libxml2 libxml2-devel libxslt libxslt-devel perl perl-ExtUtils-Embed -y  
2. 从官方获取源码包并对其解压  
> cd /usr/local/src/  
> wget https://nginx.org/download/nginx-1.14.2.tar.gz  
> tar xf nginx-1.12.2.tar.gz  
> cd nginx-1.12.2/  
3. 编译是为了检查系统环境是否符合编译安装要求，比如是否缺gcc编译器,是否支持编译参数当中的模块，并根据开启的参数等生成Makefile文件为下一步做准备；   
- configure 可执行命令,可执行程序  
- configure --help 查看可以安装的模块或者功能；查看时看到without是模块默认关掉，with是默认模块开启的    
- ssl 是支持证书模块的，sub是做监控的  
 ```bash
 ./configure --prefix=/apps/nginx \
--user=nginx \
--group=nginx \
--with-http_ssl_module \
--with-http_v2_module \
--with-http_realip_module \
--with-http_stub_status_module \
--with-http_gzip_static_module \
--with-pcre \
--with-stream \
--with-stream_ssl_module \
--with-stream_realip_module
 ```
4. 编译安装  
> make #编译步骤，根据Makefile文件生成相应的模块
> make install #创建目录，并将生成的模块和文件复制到相应的目录：  
>
> 5. 创建nginx 的用户,并修改安装目录属性；  
> useradd nginx -s /sbin/nologin -u 2000  
> chown nginx.nginx -R /apps/nginx/  
> ### nginx完成安装以后，有四个主要的目录   
> - conf:：保存nginx所有的配置文件，其中nginx.conf是nginx服务器的最核心最主要的配置文件，其他的.conf则是用来配置nginx相关的功能的，例如fastcgi功能使用的是fastcgi.conf和fastcgi_params两个文件，配置文件一般都有个样板配置文件，是文件名.default结尾，使用的使用将其复制为并将default去掉即可;  
> - html目录中保存了nginx服务器的web文件，但是可以更改为其他目录保存web文件,另外还有一个50x的web文件是默认的错误页面提示页面。  
> - logs：用来保存nginx服务器的访问日志错误日志等日志，logs目录可以放在其他路径，比如/var/logs/nginx里面。   
> - sbin：保存nginx二进制启动脚本，可以接受不同的参数以实现不同的功能。  
> 6. 查看帮助  
```bash
[root@s1 ~]# # /apps/nginx/sbin/nginx/nginx -h  
nginx version: nginx/1.12.2  
Usage: nginx [-?hvVtTq] [-s signal] [-c filename] [-p prefix] [-g directives]  
Options:  
-?,-h : this help  
-v : show version and exit
-V : show version and configure options then exit #显示版本和编译参数
-t : test configuration and exit #测试配置文件是否异常
-T : test configuration, dump it and exit #测试并打印
-q : suppress non-error messages during configuration testing #静默模式
-s signal : send signal to a master process: stop, quit, reopen, reload #发送信
号
-p prefix : set prefix path (default: /usr/share/nginx/) #指定Nginx 目录
-c filename : set configuration file (default: /etc/nginx/nginx.conf) #配置文件路径
-g directives : set global directives out of configuration file #设置全局指令
```
7. 验证版本及编译参数  
```bash
[root@s2 nginx-1.12.2]# /apps/nginx/sbin/nginx -V
nginx version: nginx/1.12.2
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-36) (GCC)
built with OpenSSL 1.0.2k-fips 26 Jan 2017
TLS SNI support enabled
configure arguments: --prefix=/apps/nginx --user=nginx --group=nginx --withhttp_ssl_module --with-http_v2_module --with-http_realip_module --withhttp_stub_status_module --with-http_gzip_static_module --with-pcre --with-stream --
with-stream_ssl_module --with-stream_realip_module
```
8. 访问编译安装的nginx web界面  
- 在网页输入**192.168.1.28** 就能限制主站点   
9. 创建Nginx自启动脚本；  
- 没有设置自启动脚本之前，启动方式  
> /apps/nginx/sbin/nginx 启动服务   
> /apps/nginx/sbin/nginx -s stop 关闭服务   
> /apps/nginx/sbin/nginx -s reload 重新加载   
>
> - 从yum安装里拷贝文件  
```bash
[root@CentOS7 ~]#cat /usr/lib/systemd/system/nginx.service 
[Unit]
Description=The nginx HTTP and reverse proxy server
After=network.target remote-fs.target nss-lookup.target

[Service]
Type=forking
PIDFile=/apps/nginx/logs/nginx.pid
# Nginx will fail to start if /run/nginx.pid already exists but has the wrong
# SELinux context. This might happen when running `nginx -t` from the cmdline.
# https://bugzilla.redhat.com/show_bug.cgi?id=1268621
ExecStartPre=/usr/bin/rm -f /apps/nginx/logs/nginx.pid
ExecStartPre=/apps/nginx/sbin/nginx -t
ExecStart=/apps/nginx/sbin/nginx
ExecReload=/bin/kill -s HUP $MAINPID
ExecRestop=/bin/kill -s TERM $MAINPID
#KillSignal=SIGQUIT1
#TimeoutStopSec=5
KillMode=process
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```
- 写好启动脚本以后启动方式  
> systemctl start nginx 启动服务   
> systemctl stop nginx 关闭服务    
> systemctl reload nginx 重新加载  
- 可以通过进程启动时间来判断重新加载成功没有，其中主进程时间没有变化，**工作进程发生变化**。  
- 将/apps/nginx/sbin/nginx 设置为软连接到/usr/sbin/下  
> ln -sv /apps/nginx/sbin/nginx /usr/sbin/  
> nginx -t 可以通过缩写开启关闭或者重新加载启动脚本  
> nginx -s reload 重新加载nginx  
10. 在nginx集群里面所有的文件里面nginx用户必须是同一个用户，并且用户的Pid和Uid都要相同，要是不同就删除重新创建用户；  
- 实例：在生产环境当中，一个集群里面两台nginx使用的用户不同（a和b），一个人上传几张图片到blog网站上去，第一张走的a服务，第二张走的b服务，上传到网站后，别人访问只能访问其中一个服务上传的图片，因为不同图片所属主可能不一样。  
- 用户nginx的nginx不一样  
```bash 
systemctl stop nginx   #先停止服务
userdel -rf nginx   #强制删除用户  
useradd nginx -u 2001   #创建用户  
getent passwd nginx  #查看用户
nginx:x:2000:2000::/home/nginx:/sbin/nologin
id nginx  #查看用户uid和pid
uid=2000(nginx) gid=2000(nginx) groups=2000(nginx)
```
## 配置文件   
- yum安装主配置文件存放位置  
> ll /etc/nginx/nginx.conf  
- yum安装默认站点位置  
> ll /usr/share/nginx/html/  
- 源码安装主配置文件存放位置  
> ll /apps/nginx/conf/nginx.conf  
- 源码安装主站点存放位置   
> ll /apps/nginx/html/index.html   
- **nginx设置内核个数，会比实际内核少一个，使用top命令查看内核个数，其中0核空出来给其他服务使用，1核开始给nginx使用，（n-1）；**  
1. 启动Nginx工作进程的用户和组,需要修改 /data/和/apps/nginx 的属性  
> user nginx nginx;  
2. 进程数和使用的事件驱动  
```bash
events {
    worker_connections  1024;
    use epoll;
    accept_mutex on;    #同一时刻只有一个请求避免多个睡眠进程被唤醒
    multi_accept on;   #每个工作进程可以接受多个新的网络连接，，此指令默认为关闭，即默认为一个工作进程只能一次接受一个新的网络连接。 
}
```
3. 使用子配置文件，在主配置文件最后定义自配置文件位置，**放在http配置下** 
> include /apps/nginx/conf.d/*.conf  
4. nginx设置内核个数，会比实际内核少一个，使用top命令查看内核个数，其中**0**核空出来给其他服务使用，1核开始给nginx使用，（n-1）；   
5. pid保存文件路径，其中logs要有写权限，不然pid，不会生成。是主进程的进程号。  
> pid /apps/nginx/logs/nginx.pid;   
### http详情配置，http是可以全局生效的，server是单个里面生效。  
6. nginx不是使用后缀辨别文件类型的，使用的MIME类型（国际标准）来辨别的。  
> include mime.types; #导入支持的文件类型  
> default_type application/octet-stream; #设置默认的类型，会提示下载不匹配的类型文件  
> ll /apps/nginx/conf/mime.types  
7. sendfile on; 指定是否使用sendfile系统调用来传输文件,sendfile系统调用在两个文件描述符之间直接传递数据(完全在内核中操作)，从而避免了数据在内核缓冲区和用户缓冲区之间的拷贝，操作效率很高，被称之为零拷贝，硬盘 >> kernel buffer (快速拷贝到kernelsocket buffer) >>协议栈。  
8. 设置会话时间
- 设置会话保持时间，65才是决定的时间，60是告诉浏览器响应的时间，浏览器收到服务器返回的报文，可以设置成一样  
> keepalive_timeout 65 60;   
- 设定保持连接超时时长，0表示禁止长连接，默认为75s，通常配置在http字段作为站点全局配置 keepalive_requests number; #在一次长连接上所允许请求的资源的最大数量，默认为100次
> keepalive_requests 3;
8. 压缩  
9. 保持会话  
10. 可以修改错误页面，不让别人访问出错，看到nginx错误页面，修改成自己设定的页面，修改50x.html文件就可以。可以填写404，403等。  
```
ll /apps/nginx/html/50x.html
error_page   500 502 503 504  /50x.html;
    location = /50x.html {
    root   html;
    }   

```
### 设置web页面，server 访问的域名，地址等配置  
11. nginx，一个IP可以设置多个域名，生成独立的站点  
- 第一个站点,在主配置文件里面修改  
```bash  
 server {
    listen       80;    #设置监听地址端口
    server_name www.xiangzi.com;   #设置server name，可以以空格隔开写多个并支持正则表达式，(单机需要在window，C:\Windows\System32\drivers\etc修改解析)
    charset utf-8;  #设置编码格式，默认俄语，修改成支持中文，utf-8;    
# 主站点    
    location / {
            root   html;
            index  index.html index.htm;
        }
    error_page 500 502 503 504 /50x.html; #定义错误页面
        location = /50x.html {
        root html;
        }
# 第二个连接，需要在/data/下创建/llx/index.html,才能访问。
# 在网页输入192.168.1.28/llx，就可以访问。http://www.xiangzi.com/llx/
        location /llx {
            root   /data/;
            index  index.html index.htm;
        }
# 第三个连接，需要在/data下创建/dyj/index.html,才能访问。
# 在网页输入192.168.1.28/dyj，就可以访问，http://www.xiangzi.com/dyj/
        location /dyj {
            root   /data/;
            index  index.html index.htm;
        }
    }
```
- 第二、三站点，创建两个子配置文件，在里面创建。基于不同的IP、不同的端口以及不用得域名实现不同的虚拟主机，依赖于核心模块ngx_http_core_module实现。  
- **指定web的家目录，在定义location的时候，文件的绝对路径等于 root+location**  
- 创建一个pc端和mobile端
```bash
mkdir /apps/nginx/conf/conf.d/   #
mkdir /apps/nginx/conf/conf.d/pc.conf #创建自配置文件
mkdir /data/nginx/html/pc -p
echo "pc web" > /data/nginx/html/pc/index.html
cat pc.conf 
server  {
	listen	80;
	server_name www.xiangzi.net;

	location / {
		root	/data/nginx/html/pc;
	    index index.html;
	}
}

mkdir /apps/nginx/conf/conf.d/mobile.conf
mkdir /data/nginx/html/mobile -p
echo "mobile web" >> /data/nginx/html/mobile/index.html
cat /apps/nginx/conf/conf.d/mobile.conf 
server  {
	listen	80;
	server_name mobile.xiangzi.com;

	location / {
		root	/data/nginx/html/mobile;
	    index index.html;
	}
}
nginx -t   #检查配置文件语法对没有  
nginx -s reload  #重新加载
```
- 浏览器访问输入www.xiangzi.net  
- moblile.xiangzi.com
12. 在上面创建第二个web界面的基础上，在创建子文件  
- 创建的image  
```bash
mkdir /data/nginx/html/pc/image
ls
aa.jpg  Aa.jpg  Aa.JPG
cat pc.conf 
server  {
	listen	80;
	server_name www.xiangzi.net;

	location / {
		root	/data/nginx/html/pc;
	    index index.html;
	}

	location /imgae {
		root	/data/nginx/html/pc;
	    index index.html;
	}
}
nginx -t
nginx -s reload
```
- 访问输入 www.xiangzi.net/image/aa.jpg  
13. root与alias： 
- root：指定web的家目录，在定义location的时候，文件的绝对路径等于 root+location  
- alias：定义路径别名，会把访问的路径重新定义到其指定的路径,/about,可以使用正则表达式，将输错的url都重定向到主页面；   
```bash 
server {
    listen 80;
        server_name www.magedu.net;
        location / {
        root /data/nginx/html/pc;
    }
    location /about { #使用alias的时候uri后面如果加了斜杠则下面的路径配置必须加斜杠，否则403
    alias /data/nginx/html/pc; #当访问about的时候，会显示alias定义的/data/nginx/html/pc
里面的内容。
    index index.html;
    }
}
nginx -t 
nginx -s reload
```
14. location的详细使用 
- 在没有使用正则表达式的时候，nginx会先在server中的多个location选取匹配度最高的一个uri，uri是用户请求的字符串，即域名后面的web文件路径，然后使用该location模块中的正则url和字符串，如果匹配成功就结束搜索，并使用此location处理此请求。  
- uri  是用户写域名后的内容  
```bash
语法规则： location [=|~|~*|^~] /uri/ { … }
= #用于标准uri前，需要请求字串与uri精确匹配，如果匹配成功就停止向下匹配并立即处理请求。
~ #用于标准uri前，表示包含正则表达式并且区分大小写
~* #用于标准uri前，表示包含正则表达式并且不区分大写
!~ #用于标准uri前，表示包含正则表达式并且区分大小写不匹配
!~* #用于标准uri前，表示包含正则表达式并且不区分大小写不匹配
^~ #用于标准uri前，表示包含正则表达式并且匹配以什么开头
$ #用于标准uri前，表示包含正则表达式并且匹配以什么结尾
\ #用于标准uri前，表示包含正则表达式并且转义字符。可以转. * ?等
* #用于标准uri前，表示包含正则表达式并且代表任意长度的任意字符
```
- 精确加密：在server部分使用location配置一个web界面，要求：当访问nginx 服务器的/login的时候要显示指定html文件的内容；  
- uri区分大小写   
> location ~ /A.?\.jpg #只匹配A.*.jpg,写出来的字母都已经固定的内容。   
- 模糊匹配  
- uri不区分大小写  
- 对用户请求的uri做模糊匹配，也就是uri中无论都是大写、都是小写或者大小写混合，此模式也都会匹配，通常使用此模式匹配用户request中的静态资源并继续做下一步操作。  
- **图片和静态资源在nginx中是挂载储存在单独的服务器上，PHP不处理图片大资源**  
>  location ~* /aa.jpg   
> location ~* \.(gif|jpg|jpeg|bmp|png|tiff|tif|ico|wmf|js)$
- 对于不区分大小写的location，则可以访问任意大小写结尾的图片文件,如区分大小写则只能访问aa.jpg，不区分大小写则可以访问aa.jpg以外的***已有的***资源比如Aa.JPG、aA.jPG这样的混合名称文件。  
#### 通常情况都是不区分大小写 作为web服务器 
- 文件后缀,static 静态  
- 一旦匹配jpg 等后缀的，就直接进static 目录下  
- nginx挂载到一个服务器专门存放图片的,所有nginx都从静态服务器里面共享图片。
```bash
    location ~* \.(gif|jpg|jpeg|bmp|png|tiff|tif|ico|wmf|js)$ {
    root /data/nginx/images1;
    index index.html;
    }
```

- 实现动静分离，static下是静态的
```bash
server  {
    listen  80; 
    server_name www.xiangzi.net;

    location / { 
        root    /data/nginx/html/pc;
        index index.html;
    }      
  
    location ^~ /static {   静态库                   
        root    /data/nginx/html/pc;
        index index.html;
    }   
}
```
- 匹配优先级   
- 匹配优先级：=, ^~, ～/～*，/location优先级：(location =) > (location 完整路径) > (location ^~ 路径) > (location~,~* 正则顺序) > (location 部分起始路径) > (/)  
```bash
    location ^~ /images {
    root /data/nginx;
    index index.html;
    }
    location /images1 {
    root /data/nginx/html/pc;
    index index.html;
    }
重启Nginx并访问测试，实现效果是访问images和images1返回不同的结果
[root@s2 images]# curl http://www.magedu.net/images1/
pc web
[root@s2 images]# curl http://www.magedu.net/images1/
pc web
```
#### 生产使用案例:  
```bash  
#直接匹配网站根会加速Nginx访问处理:
location = / {
......;
}
location / {
......;
}
#静态资源配置：
location ^~ /static/ {
......;
}
# 或者
location ~* \.(gif|jpg|jpeg|png|css|js|ico)$ {
......;
}
#多应用配置
location ~* /app1 {
......;
}
location ~* /app2 {
......;
}
```
15. 基于认证服务的，网络控制（一般不用，在防火墙就阻止了）   
- 访问控制基于模块ngx_http_access_module实现，可以通过匹配客户端源IP地址进行限制。  
```bash 
location = /login/ {
root /data/nginx/html/pc;
}
location /about {
alias /data/nginx/html/pc;
index index.html;
deny 192.168.1.1;
allow 192.168.1.0/24;
allow 10.1.1.0/16;
allow 2001:0db8::/32;
deny all; #先允许小部分，再拒绝大部分
}
```
16. Nginx账户认证功能  
- 访问一个url的时候就需要账户密码登录,一些页面需要加密。  
- 设置账户密码  
```bash
mkdir /data/nginx/html/pc/login
echo login > /data/nginx/html/pc/login/index.html
yum install httpd-tools.x86_64 -y   #生成密码的工具
htpasswd -h  #查看帮助
htpasswd -cbm /apps/nginx/conf/.htpasswd user1 123456  #-c创建目录
Adding password for user user1
htpasswd -bm /apps/nginx/conf/.htpasswd user2 123456 #创建第二个用户需要去除-c,不然要覆盖第一个文件。
Adding password for user user2
tail /apps/nginx/conf/.htpasswd #查看创建的文件
user1:$apr1$K87TJERU$Ev0kuRahTrFUMbbNWqIjw/
user2:$apr1$FnYQR7zm$/tDP1xCrPHm9VFwCDGU9m0
vim /apps/nginx/conf/conf.d/pc.conf #修改配置文件

    location = /login/ {
        root /data/nginx/html/pc;
        index index.html;
    auth_basic "login password";   #login设置账户密码
    auth_basic_user_file /apps/nginx/conf/.htpasswd; #账户密码啊存放位置
    }
```
17. 自定义错误页面
- 定义pc端web服务错误    
```bash
mkdir /data/nginx/html/pc/error.html  
echo pc error page > /data/nginx/html/pc/index.html
vim  /apps/nginx/conf/conf.d/pc.conf

error_page 500 502 503 504 404 /error.html;
    location = /error.html {
        root /data/nginx/html/pc;
    }
nginx -t
nginz -s reload
```
- 用域名访问错误的地方   
18. 自定义访问日志      
- 日志需要按时去打包。清理不然文件太大会导致存储问题   
- 大量日志解决办法，不能直接删除，需要先清空   
> echo  > access.log   
> rm -rf access.log   
- 可以写脚本，大于多少就自动打包，然后清空生成新的文件。
- 日志不会清零。需要运维工程师自己清空。不能删除，删除就不能产生数据了。需要重启，或者重新加载才能生成。 
> ll /apps/nginx/logs  
- 访问日志和错误日志分开存放，每个都是独立的   
- 对 web日志  分离保存
```bash
vim  /apps/nginx/conf/conf.d/pc.conf 
 access_log /apps/nginx/logs/www_xiangzi_net_error.log;            access_log /apps/nginx/logs/www_xiangzi_net_access.log;
nginx -t
nginx -s reload
ls /apps/nginx/logs/         #生成日志文件
access.log  nginx.pid                   www_xiangzi_net_error.log
error.log   www_xiangzi_net_access.log

```
19. ：检测文件是否存在
- 将用户访问的uri，不存在的重定向到其他地方地址（跳转到首页）   
```bash
mkdir /data/nginx/html/pc/about
echo "default" >> /data/nginx/html/pc/about/default.html
vim  /apps/nginx/conf/conf.d/pc.conf
   location /about {
        root /data/nginx/html/pc;
        #alias /data/nginx/html/pc;
        index index.html;
        #try_files $uri /about/default.html;
try_files $uri $uri/index.html $uri.html /about/default.html; # 可以设置几个跳转几个网页，第一个找不到，找第二个，第二个找不到找，找第三个，再没有。就选择所默认的。
        try_files $uri $uri/index.html $uri.html =777;               
# 要是几个默认文件都没有，就报最外面的自定义的错误页面。
access_log /apps/nginx/logs/www_xiangzi_net_login.log access_json;     #日志放在特定的位置，生成特定的日志记录     
    }   
nginx -t
nginx -s reload
```
20. 下载服务器配置  
- 下载服务器像aliyuan镜像源一样的下载方式  
```bash
mkdir /data/nginx/html/pc/download #下载不需要创建idnex.html文件
vim /apps/nginx/conf/conf.d/pc.conf
   location /download {
        autoindex on; #自动索引功能
        autoindex_exact_size on; #这一项一般关掉，会消耗计算机cpu,粗略计算即可，计算文件确切大小（单位bytes），off只显示大概大小（单位kb、mb、gb）

        autoindex_localtime   on;  #显示本机时间而非GMT(格林威治)时间
        limit_rate 10k;  #允许下载最大速度为10k,; #限制响应给客户端的传输速率，单位是bytes/second，默认值0表示无限制限速与不限速的对比：
        root /data/nginx/html/pc;
    }                 
nginx -t
ngixn -s reload
```
- 访问网站 http://www.xiangzi.net/download/    
- 可以让开发设置好看的网页，也可以自己去GitHub上看  
21. 上传服务器  
- 上传放主配置文件里面，一些用户都要使用上传。  
```bash
client_max_body_size 10m；
client_body_buffer_size 16k;
client_body_temp_path /apps/nginx/temp 1 2 2;
```
- reload nginxh会自动创建temp.  
22. 其他配置  
- 对那种浏览器禁用长连接，一般会禁止IE浏览器，因为windows的浏览器访问很麻烦。    
> keepalive_disable msie[1-6];  
>
> - 访问限制，除了GET之外其它方法仅允许192.168.76.0/24网段主机使用  
- 上传一般直接在防火墙就进行限制,不要等到Nginx的时候才限制了,不然经过负载均衡和防火墙会消耗资源;
```bash
mkdir /data/nginx/html/pc/upload
echo upload > /data/nginx/html/pc/upload/index.html
vim /apps/nginx/conf/conf.d/pc.conf
    location /upload {
        root /data/magedu/pc;
        index index.html;
        limit_except GET {
        allow 192.168.76.0/24;
        deny all;
    }
    }
nginx -t
nginx -s reload
```
- directio size | off; #操作完全和aio相反，aio是读取文件而directio是写文件到磁盘，启用直接I/O，默认为关闭，当文件大于等于给定大小时，例如directio 4m，同步（直接）写磁盘，而非写缓存。  
- 缓存，很多公司都没有配置，配置了更好。  
```bash
# 最大缓存10000，每次访问时长有20秒，在每30秒内访问同一个文件两次，就缓存文件
    open_file_cache      max=1000 inactive=20s;
    open_file_cache_valid  30s;
    open_file_cache_min_uses 2;                                       open_file_cache_errors on; 
```

## Nginx高级配置  
1. ：Nginx 状态页：  
- 基于nginx模块ngx_http_auth_basic_module实现，在编译安装nginx的时候需要添加编译参数--withhttp_stub_status_module，否则配置完成之后监测会是提示语法错误。   
```bash
配置示例：
    location /nginx_status {
        stub_status;
        allow 192.168.0.0/16;
        allow 172.20.0.0/16;
        allow 127.0.0.1;
        deny all;
}
nginx -t
nginx -s reload
# 在网页输入 www.xiangzi.net/nginx_status  显示
Active connections: 2 
server accepts handled requests
 129 129 242      #三个数字分别对应accepts,handled,requests三个值
Reading: 0 Writing: 1 Waiting: 1 
```
- Active connections： 当前处于活动状态的客户端连接数，包括连接等待空闲连接数。
- accepts：统计总值，Nginx自启动后已经接受的客户端请求的总数。
- handled：统计总值，Nginx自启动后已经处理完成的客户端请求的总数，通常等accepts,除非有因
- worker_connections限制等被拒绝的连接。
- requests：统计总值，Nginx自启动后客户端发来的总的请求数。
- Reading：当前状态，正在读取客户端请求报文首部的连接的连接数。
- Writing：当前状态，正在向客户端发送响应报文过程中的连接数。
- Waiting：当前状态，正在等待客户端发出请求的空闲连接数，开启 keep-alive的情况下,这个值等于active – (reading+writing).  
- **客服端连接数8000，nginx能抗住（能支持两三万的高并发），但是后方服务端不一定能抗住，用这个状态页面值，做一个监控趋势图，方便查看**  
2. Nginx 第三方模块  
- 第三模块是对nginx 的功能扩展，第三方模块需要在编译安装Nginx 的时候使用参数--add-module=PATH指定路径添加，有的模块是由公司的开发人员针对业务需求定制开发的，有的模块是开源爱好者开发好之后上传到github进行开源的模块，nginx支持第三方模块需要从源码重新编译支持，
- 开源的echo模块 https://github.comopenrestyecho-nginx-module；  
```bash 
systemctl stop nginx
vim /apps/nginx/conf/conf.d/pc.conf
 location /echo {
        default_type text/html;
        echo hello;
        echo world;
    }

    location /main {
        index index.html;
        default_type text/html;
        echo "hello world,main-->";
        echo_reset_timer;
        echo_location /sub1;
        echo_location /sub2;
        echo "took $echo_timer_elapsed sec for total.";
        }
    location /sub1 {
        echo_sleep 1;
        echo sub1;
        }
    location /sub2 {                                                            
        echo_sleep 1;
        echo sub2;
        }
nginx -t
nginx: [emerg] unknown directive "echo_reset_timer" in
/apps/nginx/conf/conf.d/pc.conf:86
nginx: configuration file /apps/nginx/conf/nginx.conf test failed
# 解决以上报错问题  
nginx -s stop    #加新的模块需要停服务
cd /usr/local/src  
yum install git -y   #需要安装 git工具
git clone https://github.com/openresty/echo-nginx-module.git #从网上git开源恶臭.
ll /usr/local/src/echo-nginx-module/src  # 查看模块
cd /usr/local/src/nginx-1.14.2/ #进入安装文件进行编译；
nginx -V   查看上次编译安装的包，复制下来，准备在重新添加模块
./configure      #重新安装编译
--prefix=/apps/nginx \
--user=nginx --group=nginx \
--with-http_ssl_module \
--with-http_v2_module \
--with-http_realip_module \
--with-http_stub_status_module \
--with-http_gzip_static_module \
--with-pcre \
--with-stream \
--with-stream_ssl_module \
--with-stream_realip_module \
--with-http_perl_module \
--add-module=/usr/local/src/echo-nginx-module
make && make install    安装编译
/apps/nginx/sbin/nginx -t  #检查语法通过
/apps/nginx/sbin/nginx #开启 nginx
curl www.xiangzi.net/echo   #重新测试访问
hello
world
curl www.xiangzi.net/main  重新测试访问
hello world,main-->
sub1
sub2
took 2.004 sec for total.  
```

3. Nginx 变量使用：  
- 存放了客户端的地址，注意是客户端的公网IP，也就是一家人访问一个网站，则会显示为路由器的公网IP.查看本机出口地址 IP     
> $remote_addr;  
> 192.168.1.1  
- 变量中存放了URL中的指令，例如http://www.magedu.net/main/index.do?  
> $args；  
> id=20190221&partner=search中的id=20190221&partner=search  
- 保存了针对当前资源的请求的系统根目录  
> $document_root；  
> /apps/nginx/html   
- 保存了当前请求中不包含指令的URI，注意是不包含请求的指令  
> $document_uri；  
>  /echo
> http://www.magedu.net/main/index.html会被定义为/main/index.html
- 存放了请求的host名称。   
> $host；  
> www.xiangzi.net
- 客户端浏览器的详细信息,访问服务器的浏览器  
> $http_user_agent； 
> Mozilla/5.0 (Windows NT 10.0; Win64; x64)AppleWebKit/537.36 (KHTML, like Gecko) Chrome/73.0.3683.86 Safari/537.36 .Google浏览器   
- 如果nginx服务器使用limit_rate配置了显示网络速率，则会显示，如果没有设置， 则显示0。  
> limit_rate 10240;
> echo $limit_rate;
> echo 10240  
- 客户端请求Nginx服务器时随机打开的端口，这是每个客户端自己的端口   
> $remote_port；   
- 12368  
- 请求资源的方式，GET/PUT/DELETE等  
> $request_method；  
> GET 
- 当前请求的资源文件的路径名称，由root或alias指令与URI请求生成的文件绝对路径，
> $request_filename；
>/apps/nginx/html/echo
- 包含请求参数的原始URI，不包含主机名，如：/main/index.html
> $request_uri；
> /echo
- 请求的协议，如ftp，https，http等。
> $scheme；
> http
- 保存了客户端请求资源使用的协议的版本，如HTTP/1.0，HTTP/1.1，HTTP/2.0等。
> $server_protocol；
> HTTP/1.1
- 保存了服务器的IP地址。
> $server_addr；
> 192.168.1.28
- 请求的服务器的主机名。
> $server_name；
> www.xiangzi.net
- 请求的服务器的端口号。
> $server_port；
> 80

```bash
   location /echo {
        default_type text/html;
        #echo hello;
        #echo world;              
    echo    $remote_addr;
    echo    $args;
    echo    $document_root;
    echo    $document_uri;
    echo    $host;
    echo    $http_user_agent;
    echo    $http_cookie;
            limit_rate 10240;
    echo    echo $limit_rate;
    echo    $remote_port;
    echo    $remote_user;
    echo    $request_body_file;
    echo    $request_method;
    echo    $request_filename;
    echo    $request_uri;
    echo    $scheme;
    echo    $server_protocol;
    echo    $server_addr;
    echo    $server_name;
    echo    $server_port;
    }
```
4. 自定义变量  
- 假如需要自定义变量名称和值，使用指令set $variable value;  
- 通过set 设置变量，再用echo打印出来。  
- 用set,设置内置变量,在打印出来
```bash
set $name magedu;
echo $name;
set $my_port $server_port;
echo $my_port;
echo "$server_name:$server_port";
```
5. 自定义访问日志  
- Nginx一般错误日志只有一个，但是访问日志可以在不同的server中定义多个，定义一个日志需要使用access_log指定日志的保存路径，使用log_format指定日志的格式，格式中定义要保存的具体日志内容。  
- 自定义json格式日志   
- Nginx 的默认访问日志记录内容相对比较单一，默认的格式也不方便后期做日志统计分析，生产环境中通常将nginx日志转换为json日志，然后配合使用ELK做日志收集-统计-分析。
```bash
vim /apps/nginx/conf/nginx.conf
   log_format access_json '{"@timestamp":"$time_iso8601",'
    '"host":"$server_addr",'
    '"clientip":"$remote_addr",'
    '"size":$body_bytes_sent,'
    '"responsetime":$request_time,'
    '"upstreamtime":"$upstream_response_time",'
    '"upstreamhost":"$upstream_addr",'
    '"http_host":"$host",'
    '"uri":"$uri",'
    '"domain":"$host",'
    '"xff":"$http_x_forwarded_for",'
    '"referer":"$http_referer",'
    '"tcp_xff":"$proxy_protocol_addr",'
    '"http_user_agent":"$http_user_agent",'
    '"status":"$status"}';
    access_log /apps/nginx/logs/access_json.log access_json;
#对子配置文件进行开启日志
vim /apps/nginx/conf/conf.d/pc.conf
    access_log /apps/nginx/logs/www_xiangzi_net_access.log access_json;
开启log ，  记录在www_xiangzi_net_access.log下，用access_json形式的格式记录
# 保存日志文件到指定路径并进测试：
 tail /apps/nginx/logs/www_xiangzi_net_access.log  #查看日志
{"@timestamp":"2019-06-01T19:59:25+08:00","host":"192.168.1.28","clientip":"192.168.1.1","size":14,"responsetime":0.000,"upstreamtime":"-","upstreamhost":"-","http_host":"www.xiangzi.net","uri":"/error.html/index.html","domain":"www.xiangzi.net","xff":"-","referer":"http://www.xiangzi.net/static/aa.jpg","tcp_xff":"","http_user_agent":"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/73.0.3683.86 Safari/537.36","status":"200"}
可以通过网上搜索在线JSON查看json日志是否有问题。  
```
###json格式的日志访问统计   
- 写脚本统计 
```bash
#!/usr/bin/env python
#coding:utf-8
status_200= []
status_404= []
with open("access_json.log") as f:
    for line in f.readlines():
        line = eval(line)
        if line.get("status") == "200":
            status_200.append(line.get)
        elif line.get("status") == "404":
            status_404.append(line.get)
        else:
            print("状态码 ERROR")
f.close()

print "状态码200的有--:",len(status_200)
print "状态码404的有--:",len(status_404)
#保存日志文件到指定路径并进测试：
python nginx_json.py
状态码200的有--: 1910
状态码404的有--: 13

```
- 查看文件别直接用**vim**打开，会把系统搞崩溃的，用tail 命令查看

7.Nginx的压缩功能  
- Nginx支持对指定类型的文件进行压缩然后再传输给客户端，而且压缩还可以设置压缩比例，压缩后的文件大小将比源文件显著变小，这样有助于降低出口带宽的利用率，降低企业的IT支出，不过会占用相应的CPU资源。  
- Nginx对文件的压缩功能是依赖于模块ngx_http_gzip_module  
- 压缩比等级【1—9】，一般设置在3~5等级，一般压缩静态文件，文本或者图片,视频文件等大文件不建议视频。  
```bash  
#启用或禁用gzip压缩，默认关闭
gzip on | off;
#压缩比由低到高从1到9，默认为1
gzip_comp_level level;
#禁用IE6 gzip功能
gzip_disable "MSIE [1-6]\.";
#gzip压缩的最小文件，小于设置值的文件将不会压缩
gzip_min_length 1k;
#启用压缩功能时，协议的最小版本，默认HTTP/1.1
gzip_http_version 1.0 | 1.1;
#指定Nginx服务需要向服务器申请的缓存空间的个数*大小，默认32 4k|16 8k;
gzip_buffers number size;
#指明仅对哪些类型的资源执行压缩操作；默认为gzip_types text/html，不用显示指定，否则出错
gzip_types mime-type ...;
#如果启用压缩，是否在响应报文首部插入“Vary: Accept-Encoding”
gzip_vary on | off;
```
- 测试压缩功能  
```bash
cat /etc/fstab  > /data/nginx/html/pc/test.html  #小于1k
ll /data/mginx/html/pc/test.html/Aa.jpg   #大于1k
vim /apps/nginx/conf/nginx.conf
    gzip on; 
    gzip_comp_level 5;
    gzip_min_length 1k; 
    gzip_types text/plain application/javascript application/x-javascript text/cssappl ication/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png;                                         
    gzip_vary on; 
nginx -t
nginx -s reload
#访问测试：
 curl --head --compressed http://www.xiangzi.net/test.html #文件小于1k，不用压缩传输 
 crul --head --compressed htt[://www.xiangzi.net/Aa.jpg  #文件大于1k,需要压缩，用gzip压缩传送的；
# 使用网页打开 htt[://www.xiangzi.net/Aa.jpg，用fn+f12,打开可以查看大小。明显看到压缩了。
 ETag: W/"5ca4acf8-14bd8" 数据发生变化，公司服务器数据也要发生变化。
```
### https 功能  
1. Web网站的登录页面都是使用https加密传输的，加密数据以保障数据的安全，HTTPS能够加密信息，以免敏感信息被第三方获取，所以很多银行网站或电子邮箱等等安全级别较高的服务都会采用HTTPS协议，HTTPS其实是有两部分组成：HTTP + SSL / TLS，也就是在HTTP上又加了一层处理加密信息的模块。服务端和客户端的信息传输都会通过TLS进行加密，所以传输的数据都是加密后的数据。   
```
https 实现过程如下：
1.客户端发起HTTPS请求：
客户端访问某个web端的https地址，一般都是443端口
2.服务端的配置：
采用https协议的服务器必须要有一套证书，可以通过一些组织申请，也可以自己制作，目前国内很多网站都自己
做的，当你访问一个网站的时候提示证书不可信任就表示证书是自己做的，证书就是一个公钥和私钥匙，就像一把
锁和钥匙，正常情况下只有你的钥匙可以打开你的锁，你可以把这个送给别人让他锁住一个箱子，里面放满了钱或
秘密，别人不知道里面放了什么而且别人也打不开，只有你的钥匙是可以打开的。
3.传送证书：
服务端给客户端传递证书，其实就是公钥，里面包含了很多信息，例如证书得到颁发机构、过期时间等等。
4.客户端解析证书：
这部分工作是有客户端完成的，首先回验证公钥的有效性，比如颁发机构、过期时间等等，如果发现异常则会弹出
一个警告框提示证书可能存在问题，如果证书没有问题就生成一个随机值，然后用证书对该随机值进行加密，就像
2步骤所说把随机值锁起来，不让别人看到。
5.传送4步骤的加密数据：
就是将用证书加密后的随机值传递给服务器，目的就是为了让服务器得到这个随机值，以后客户端和服务端的通信
就可以通过这个随机值进行加密解密了。
6.服务端解密信息：
服务端用私钥解密5步骤加密后的随机值之后，得到了客户端传过来的随机值(私钥)，然后把内容通过该值进行对
称加密，对称加密就是将信息和私钥通过算法混合在一起，这样除非你知道私钥，不然是无法获取其内部的内容，
而正好客户端和服务端都知道这个私钥，所以只要机密算法够复杂就可以保证数据的安全性。
7.传输加密后的信息:
服务端将用私钥加密后的数据传递给客户端，在客户端可以被还原出原数据内容。
8.客户端解密信息：
客户端用之前生成的私钥获解密服务端传递过来的数据，由于数据一直是加密的，因此即使第三方获取到数据也无
法知道其详细内容。
```
2. nginx 的https 功能基于模块ngx_http_ssl_module实现，因此如果是编译安装的nginx要使用参数ngx_http_ssl_module开启ssl功能，但是作为nginx的核心功能，yum安装的nginx默认就是开启的，编译安装的nginx需要指定编译参数--with-http_ssl_module开启  
```bash
ssl on | off;
#为指定的虚拟主机配置是否启用ssl功能，此功能在1.15.0废弃，使用listen [ssl]替代。
ssl_certificate /path/to/file;
#当前虚拟主机使用使用的公钥文件，一般是crt文件
ssl_certificate_key /path/to/file;
#当前虚拟主机使用的私钥文件，一般是key文件
ssl_protocols [SSLv2] [SSLv3] [TLSv1] [TLSv1.1] [TLSv1.2];
#支持ssl协议版本，早期为ssl现在是TSL，默认为后三个
ssl_session_cache off | none | [builtin[:size]] [shared:name:size];
#配置ssl缓存
off： 关闭缓存
none: 通知客户端支持ssl session cache，但实际不支持
builtin[:size]：使用OpenSSL内建缓存，为每worker进程私有
[shared:name:size]：在各worker之间使用一个共享的缓存，需要定义一个缓存名称和缓存空间大小，
一兆可以存储4000个会话信息，多个虚拟主机可以使用相同的缓存名称。
ssl_session_timeout time;#客户端连接可以复用ssl session cache中缓存的有效时长，默认5m
```
3. 自签名证书：   
```bash
#自签名CA证书
 cd /apps/nginx/
 mkdir certs
 cd certs/
 openssl req -newkey rsa:4096 -nodes -sha256 -keyout ca.key -x509 -days 3650 -out ca.crt #自签名CA证书
Generating a 4096 bit RSA private key
.................++
.....
Country Name (2 letter code) [XX]:CN #国家代码
State or Province Name (full name) []:BeiJing #省份
Locality Name (eg, city) [Default City]:Beijing #城市名称
Organization Name (eg, company) [Default Company Ltd]:xiangizi #公司名称
Organizational Unit Name (eg, section) []:linux #部门
Common Name (eg, your name or your server's hostname) []:www.xiangzi.ca #通用名称
Email Address []:123456@qq.com #邮箱
 ll ca.crt
-rw-r--r-- 1 root root 2102 Jun  2 08:34 ca.crt

#自制key和csr文件
 openssl req -newkey rsa:4096 -nodes -sha256 -keyout www.xiangzi.net.key -out www.xiangzi.net.csr
Generating a 4096 bit RSA private key
........................................................................++
......
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:BeiJing
Locality Name (eg, city) [Default City]:BeiJing
Organization Name (eg, company) [Default Company Ltd]:xaingzi
Organizational Unit Name (eg, section) []:linux
Common Name (eg, your name or your server's hostname) []:www.xiangzi.net
Email Address []:122342@qq.com
Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
# ll /apps/nginx/certs/
total 16
-rw-r--r-- 1 root root 2102 Jun  2 08:34 ca.crt
-rw-r--r-- 1 root root 3272 Jun  2 08:34 ca.key
-rw-r--r-- 1 root root 1744 Jun  2 08:38 www.xiangzi.net.csr
-rw-r--r-- 1 root root 3272 Jun  2 08:38 www.xiangzi.net.key

# 签发证书
openssl x509 -req -days 3650 -in www.xiangzi.net.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out www.xiangzi.net.crt

# 验证证书内容
openssl x509 -in www.magedu.net.crt -noout -text 
```
4. Nginx证书配置：
```bash
vim /apps/nginx/conf/conf.d/pc.conf
    listen 443 ssl;
        ssl_certificate /apps/nginx/certs/www.xiangzi.net.crt;
        ssl_certificate_key /apps/nginx/certs/www.xiangzi.net.key;         
        ssl_session_cache shared:sslcache:20m;
        ssl_session_timeout 10m;  
nginx -t 
nginx -s reload
```
5. Nginx支持基于单个IP实现多域名的功能.
- 还支持单IP多域名的基础之上实现HTTPS，其实是基于Nginx的SNI（Server Name Indication）功能实现，SNI是为了解决一个Nginx服务器内使用一个IP绑定多个域名和证书的功能，其具体功能是客户端在连接到服务器建立SSL链接之前先发送要访问站点的域名（Hostname），这样服务器再根据这个域名返回给客户端一个合适的证书。
```bash
#制作key和csr文件
openssl req -newkey rsa:4096 -nodes -sha256 -keyoutmobile.xiangzi.net.key -out mobile.xiangzi.net.csr
Generating a 4096 bit RSA private key
..........
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:BeiJing
Locality Name (eg, city) [Default City]:BeiJing
Organization Name (eg, company) [Default Company Ltd]:xiangzi
Organizational Unit Name (eg, section) []:python
Common Name (eg, your name or your server's hostname) []:mobile.xiangzi.net
Email Address []:3242320@qq.com
Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
#签名证书
openssl x509 -req -days 3650 -in mobile.xiangzi.net.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out mobile.xiangzi.net.crt

#验证证书内容
openssl x509 -in mobile.xiangzi.net.crt -noout -text

#Nginx 配置
cat /apps/nginx/conf/conf.d/mobile.conf
server {
    listen 80 default_server;
    server_name mobile.magedu.net;
    rewrite ^(.*)$ https://$server_name$1 permanent;
    }
server {
    listen 443 ssl;
    server_name mobile.magedu.net;
    ssl_certificate /apps/nginx/certs/mobile.magedu.net.crt;
    ssl_certificate_key /apps/nginx/certs/mobile.magedu.net.key;
    ssl_session_cache shared:sslcache:20m;
    ssl_session_timeout 10m;
    location / {
        root "/data/nginx/html/mobile";
    }
    location /mobile_status {
        stub_status;
        #allow 172.16.0.0/16;
    #deny all;
    }
}
nginx -t
nginx -s reolad
```
6. 关于favicon.ico:  
- favicon.ico 文件是浏览器收藏网址时显示的图标，当客户端使用浏览器问页面时，浏览器会自己主动发起请求获取页面的favicon.ico文件，但是当浏览器请求的favicon.ico文件不存在时，服务器会记录404日志，而且浏览器也会显示404报错。  
- 一般网页都会有小图标的，开发提供，美工设计，没有就会显示404。
- 解决办法
```bash
#一：服务器不记录访问日志：
#location = /favicon.ico {
#log_not_found off;
#access_log off;
#}
#二：将图标保存到指定目录访问：#文件名为 favicon.ico。
mkdir /apps/nginx/html/pc/images
cp windows /apps/nginx/html/pc/images/favicon.ico
vim /apps/nginx/conf/conf.d/pc.conf
    #location ~ ^/favicon\.ico$ {
        location = /favicon.ico {      
        root /data/nginx/html/pc/images;
    }
nginx -t
nginx -s reload
```
8. 隐藏nginx的版本号   
-  **server_tokens off; #隐藏Nginx server版本。**  隐藏浏览器查看nginx的版本号，防止黑客恶意攻击。 
7. 隐藏版本号，但是还能看到nginx,危险，需要更改或者隐藏服务  
 - 更改nginx源码信息并重新编译Nginx
 ```bash
 # 修改源文件
vim /usr/local/src/nginx-1.14.2/src/http/ngx_http_header_filter_module.c
    static u_char ngx_http_server_string[] = "Server: Duysjp" CRLF
将文件这一行修改成 **Duysjp**，定义响应报文中特定的server字段信息  
修改后需要重新编译
nginx -s stop
nginx -V  #复制以前编译的模块包，重新编译。
cd /usr/local/nginx-1.14.2/
./configure  --prefix=/apps/nginx --user=nginx --group=nginx --with-http_ssl_module --with-http_v2_module --with-http_realip_module --with-http_stub_status_module --with-http_gzip_static_module --with-pcre --with-stream --with-stream_ssl_module --with-stream_realip_module --add-module=/usr/local/src/echo-nginx-module
make && make install
nginx 
从浏览器查看
 ```
## Nginx Rewrite相关功能
1. ngx_http_rewrite_module模块指令：  

```bash
server {
     listen 443 ssl;
    listen 80;
         ssl_certificate /apps/nginx/certs/www.xiangzi.net.crt;
        ssl_certificate_key /apps/nginx/certs/www.xiangzi.net.key;
        ssl_session_cache shared:sslcache:20m;
        ssl_session_timeout 10m;
        server_name www.xiangzi.net;

        location / {
      root /data/nginx/html/pc;
        index index.html;
        if ($scheme = http ){ #未加条件判断，会导致死循环
        rewrite / https://www.xiangzi.net permanent;
        }
    }
}

```
- 当用户访问公司网站的时输入一个错误的URL，可以将用户重定向至官方首页。  
```bash
location / {
    root /data/nginx/html/pc;
    index index.html;
    if (!-f $request_filename) {
    #return 404 "linux";
    rewrite (.*) http://www.xiangzi.net/index.html;
    }
}
#重启Nginx并访问测试
```
- 综合，将http重定向到https，和输入错误的页面重定向到官方首页，
```bash
server {
        listen 80; 
        server_name mac.xiangzi.net;
        listen 443 ssl;
        ssl_certificate /apps/nginx/certs/mac.xiangzi.net.crt;
        ssl_certificate_key /apps/nginx/certs/mac.xiangzi.net.key;
        ssl_session_cache shared:sslcache:20m;
        ssl_session_timeout 10m;

    location / { 
        root "/data/nginx/html/mac";
        index index.html;
        if (!-f $request_filename) {
        #return 404 "linux";
        rewrite (.*) http://mac.xiangzi.net/index.html;
        }   
        if ($scheme = http ) { 
        rewrite ^(.*)$ https://$server_name$1 permanent;
            }    
    }







```