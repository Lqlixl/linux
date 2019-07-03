## 实现Nginx防盗链
1. 防盗链基于客户端携带的referer实现，referer是记录打开一个页面之前记录是从哪个页面跳转过来的标记信息，如果别人只链接了自己网站图片或某个单独的资源，而不是打开了网站的整个页面，这就是盗链，referer就是之前的那个网站域名，正常的referer信息有以下几种：
```bash
none：请求报文首部没有referer首部，比如用户直接在浏览器输入域名访问web网站，就没有referer信息。
blocked：请求报文有referer首部，但无有效值，比如为空。
server_names：referer首部中包含本主机名及即nginx 监听的server_name。
arbitrary_string：自定义指定字符串，但可使用*作通配符。
regular expression：被指定的正则表达式模式匹配到的字符串,要使用~开头，例如：
~.*\.xiangzi\.com.
# 正常通过搜索引擎搜索web网站并访问该网站的referer信息如下：
{"@timestamp":"2019-06-02T15:06:05+08:00","host":"192.168.1.28","clientip":"192.168.1.1","size":237,"responsetime":0.000,"upstreamtime":"-","upstreamhost":"-","http_host":"www.xiangzi.net","uri":"/favicon.ico","domain":"www.xiangzi.net","xff":"-","referer":"http://www.xiangzi.net/","tcp_xff":"","http_user_agent":"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/73.0.3683.86 Safari/537.36","status":"200"}
2. 实现web盗链  
- 在一个web站点盗链另一个站点的资源信息，比如图片、视频等。  
```bash
cd /apps/nginx/conf/conf.d
cp pc.conf borrow.conf
# 设置盗链web的配置文件
cat borrow.conf 
server  {
	listen	80;
	server_name www.borrow.net;
	access_log /apps/nginx/logs/borrow_access.log access_json; #错误日志也要修改
	error_log /apps/nginx/logs/borrow_error.log;
	
    location / {
		root	/data/nginx/html/borrow;
	    index index.html;
	}
}
# 准备盗链web页面  
mkdir /data/nginx/html/borrow
写页面文件
cat /data/nginx/html/borrow/index.html 
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>盗链页面</title>
</head>
<body>
<a href="http://www.xiangzi.net">测试盗链</a>
<img src="http://www.xiangzi.net/static/aa.jpg">
<img src="http://desk-fd.zol-img.com.cn/t_s720x360c5/g5/M00/03/02/ChMkJ1v9A1mIN_iKABERj1MSlcQAAtatAEvvFEAERGn385.jpg">
<img src="http://pic.netbian.com/uploads/allimg/170424/093818-1492997898b63f.jpg">
</body>
</html>
#重启Nginx并访问http://www.mageedu.net/测试
nginx -t 
nginx -s reload
#验证两个域名的日志，是否会在被盗连的web站点的日志中出现以下盗链日志信息：
tail -f /apps/nginx/logs/access_json.log
tail -f /apps/nginx/logs/borrow_access.log
#盗链成功
```
3. 实现防盗链  
- 基于访问安全考虑，nginx支持通过ungx_http_referer_module模块 https://nginx.org/en/docs/http/ngx_http_referer_module.html#valid_referers 检查访问请求的referer信息是否有效实现防盗链功能，定义方式如下：
- 除了百度，Google，等浏览器的都是盗链。
- 解决办法，在配置文件里面 ，提供图片的位置（一般是动静分离的地方），先做有效的判断，在做 if  判断，让除了有效的访问，都返回403，或者转到百度去。
```bash
# 防盗链格式
vim /apps/nginx/conf/conf.d/pc.conf
    location /images {
        root /data/nginx/html/pc;
        index index.html;
        valid_referers none blocked server_names
        *.example.com example.* www.example.org/galleries/
        ~\.google\.;
    if ($invalid_referer) {
    return 403; 
    }
#实现定义防盗链
    location ^~ /static {
            root    /data/nginx/html/pc;
            index index.html;
            valid_referers none blocked server_names *.xiangzi.comwww.xiangzi.*  ~\.google\. ~\.baidu\.;
            if ($invalid_referer) {
            return 403;
    }   
}
nginx -t
nginx -s reload
# 验证，使用浏览器访问盗链网站 www.borrow.net， 验证是否提前状态码403：

```
## Nginx 反向代理
1. 反向代理：反向代理也叫reverse proxy，指的是代理外网用户的请求到内部的指定web服务器，并将数据返回给用户的一种方式，这是用的比较多的一种方式。
- Nginx除了可以在企业提供高性能的web服务之外，另外还可以将本身不具备的请求通过某种预定义的协议转发至其它服务器处理，不同的协议就是Nginx服务器与其他服务器进行通信的一种规范，主要在不同的场景使用以下模块实现不同的功能
```bash
ngx_http_proxy_module： 将客户端的请求以http协议转发至指定服务器进行处理。
ngx_stream_proxy_module：将客户端的请求以tcp协议转发至指定服务器处理。
ngx_http_fastcgi_module：将客户端对php的请求以fastcgi协议转发至指定服务器助理。
ngx_http_uwsgi_module：将客户端对Python的请求以uwsgi协议转发至指定服务器处理
```
2. 实现htt反向代理： 
- 要求：将用户对域 www.nagedu.net 的请求转发值后端服务器处理，官方文档： https://nginx.org/en/docs/http/ngx_http_proxy_module.html  
- 环境准备  
```bash
192.168.1 28   #Nginx 代理服务器
192.168.1.38   #后端web A，Apache部署
192.168.1.48   #后端web B，Apache部署
### 部署后端Apache服务器：  
```bash
# 192.168.1.38
yum install httpd -y
echo "web1 192.168.1.38" > /var/www/html/index.html
systemctl start httpd && systemctl enable httpd
# 192.168.1.48
yum install httpd -y
echo "web2 192.168.7.48" >> /var/www/html/index.html
systemctl start httpd && systemctl enable httpd

#访问测试
curl http://192.168.1.38
    web1 192.168.7.103
curl http://192.168.1.48
    web2 192.168.7.104

```
3. Nginx反向代理设置
- 反向代理配置参数  
```bash
server {
    listen 80;
        server_name www.xiangzi.net;
    location / {
    index index.html index.php;
        root /data/nginx/html/pc;
    }
    location /app1 {
    #proxy_pass http://192.168.7.103:80/; #注意有后面的/，会固定服务器的文件。
    proxy_pass http://172.20.76.38:80;
        }
    }
    location /app1 {
    proxy_pass http://172.20.76.48:80;
        }
    }
curl -L http://172.20.76.38/app1/
web1 192.168.1.38
curl -L http://172.20.76.48/app1/
web1 192.168.1.48
```
- 隐藏头部信息  
```bash
curl -I www.xiangzi.net
    HTTP/1.1 200 OK
    Server: Duysjp
    Date: Sun, 02 Jun 2019 12:59:00 GMT
    Content-Type: text/html
    Content-Length: 3
    Last-Modified: Thu, 30 May 2019 03:11:42 GMT
    Connection: keep-alive
    Keep-Alive: timeout=65
    ETag: "5cef49ee-3"
    Accept-Ranges: bytes

# 上面头部信息都可以隐藏

 ```
4. 反向代理-缓存功能  
- 先有人访问了后端apache服务内容，然后内容缓存在nginx上。
```bash
vim /apps/nginx/conf/nginx.conf
 proxy_cache_path /apps/nginx/proxycache levels=1:2:2 keys_zone=proxycache:5    12m inactive=120s max_size=1g;
proxy_cache zone | off; 默认off
#指明调用的缓存，或关闭缓存机制；Context:http, server, location
#levels=1:2:2 #定义缓存目录结构层次，1:2:2可以生成2^4x2^8x2^8=1048576个目录
#keys_zone=proxycache:20m #指内存中缓存的大小，主要用于存放key和metadata（如：使用次数）
#inactive=120s； #缓存有效时间
#max_size=1g; #最大磁盘占用空间，磁盘存入文件内容的缓存空间最大值

vim /apps/nginx/conf/conf.d/mac.conf
   location /app1 {
        proxy_pass http://172.20.76.38:80;
        proxy_hide_header Server;
        proxy_hide_header ETag;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_cache proxycache;
        proxy_cache_key $request_uri;
        proxy_cache_valid 200 302 301 1h; 
        proxy_cache_valid any 1m; 
    }   
# 先18 机器上测试
ab -n5000 -c500 mac.xiangzi.net/app1/access_json.log
记录结果
nginx -t
ngixn -s reload
# 在此测试
ab -n5000 -c500 mac.xiangzi.net/app1/access_json.log

28
查看缓存文件
ll /apps/nginx/proxycache

```

5. Nginx http 反向代理高级应用  
- 对多台服务器进行转发。提供相应的服务器状态检测，
- Nginx可以基于ngx_http_upstream_module模块提供服务器分组转发、权重分配、状态监测、调度算法等高级功能。  
```bash
upstream app1 {   
    server 172.20.76.38:80 weight=2 max_fails=3 fail_timeout=5;
    server 172.20.76.48:80 weight=1 max_fails=3 fail_timeout=5;   
    server 172.20.76.58:80 weight=2 max_fails=3 fail_timeout=5 backup;   
}  
    server {
        listen 80;
        server_name mac.xiangzi.net;
    
    location / {
        root /data/nginx/html/mac;
        index index.html;
    }

     location /app1 {
         index index.html;
        proxy_pass http://app1;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; #添加客户端IP到报文头
部
    }  
}
nginx -t
nginx -s reload
换台机器测试
while true; do curl -L http://mac.xiangzi.net/app1/index.html; sleep 1 ;done

# 反向代理-客服端IP透传
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; #添加客户端IP到报文头

#后端web服务器配置
Apache：
vim /etc/httpd/conf/httpd.conf    修改日志
 LogFormat "\"%{X-Forwarded-For}i\  %h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{    User-Agent}i\"" combined
 #重启apache访问web界面并验证apache日志：
"172.20.76.18\   172.20.76.28 - - [04/Jun/2019:14:54:51 +0800] "GET /app1/index.html HTTP/1.0" 200 17 "-" "curl/7.29.0"
可以看到访问者的IP
```
6. 实现动静分离  
1. 环境设置 
```bash 
172.20.76.18  #NFS存储服务器  
172.20.76.28  #Nginx反向代理器
```
- 搭建
```bash
NFS的server设置，172.20.76.18
yum install nfs-utils.x86_64 -y
mkdir /data/static -p
cat /etc/exports
/data/static *(rw,no_root_squash)
ls /data/static/
1.jpg  2.jpg  3.jpg  4.jpg
systemctl start nfs
systemctl enable nfs

Nginx反向代理器 172.20.76.28
安装nfs包
yum install nfs-utils -y
查看18的共享目录
showmount -e 172.20.76.18
Export list for 172.20.76.18:
/data/static *
挂载
mount -t nfs 172.20.76.18:/data/static /data/nginx/html/mac/static
永久开机挂载，_netdev，要添加网络挂载，不然开机自动挂载可能一直在挂载失败
服务器一直卡哪儿
cat /etc/fstab 
172.20.76.18:/data/static /data/nginx/html/mac/static           nfs     defaults,**_netdev**    0 0 
mount -a
df -TH
vim /apps/nginx/conf/conf.d/mac.conf
    location /static {
        root /data/nginx/html/mac;
        index index.html;
        }
nginx -t
nginx -s reload

测试
http://mac.xiangzi.net/static/4.jpg




























































































































































































































































1.实现以upstream 分组的方式实现tcp以及http反向代理
 2.实现LNMP的单机环境，PHP要5.6或更高版本
 3.实现Nginx防盗链
 4.实现proxy_cache缓存









