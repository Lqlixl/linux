# Tomcat

## jdk

### 1.安装

```bash
#orcle 下载jdk1.8.0_191
	
#yum安装
yum install jdk-8u211-linux-x64.rpm -y

```

### 2. 全局配置

```bash
#配置变量
cat /etc/profile.d/jdk.sh
export JAVA_HOME=/usr/java/default
export PATH=$JAVA_HOME/bin:$PATH

#立即启动
source /etc/profile.d/jdk.sh

```



## Tomcat

### 1. 安装下载

```bash
#apache tomcat 下载二进制包
apache-tomcat-8.5.42.tar.gz

#解压包到/usr/local包下
tar xf apache-tomcat-8.5.42.tar.gz -C /usr/local
cd /usr/local/

#将解压的文件夹软连接成tomcat
ln -sv apache-tomcat-8.5.42/ tomcat
```



### 2. 查看设置启动程序

```bash
#查看启动方式
cd tomcat/bin/
./catalina.sh --hlep

#查看版本
./catalina.sh version

#启动tomcat服务
./catalina.sh start
或者 ./startup.sh
#查看端口

8080

8009

8005

#关闭服务
./catalina.sh stop
或者 ./shutdown.sh


#出现8005端口起不来的问题
目录：/lib/jvm/java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64/jre/lib/security/java.security

修改$JAVA_HOME/jre/lib/security/java.security 文件中 securerandom.source 配置项:
将原本的：securerandom.source=file:/dev/random 
修改为： securerandom.source=file:/dev/urandom

```

### 3 .配置

- Tomcat
- Tomcat中默认网站根目录是CATALINA_BASE/webapps/
  在Tomcat的webapps目录中，有个非常特殊的目录ROOT，它就是网站默认根目录。
  将eshop解压后的文件放到这个ROOT中。
- bbs解压后文件都放在CATALINA_BASE/webapps/bbs目录下。
  每一个虚拟主机的目录都可以使用appBase配置自己的站点目录，里面都可以使用ROOT目录作为主站目录。

```bash
#在ROOT下创建html目录
[root@centos ROOT]# pwd
/usr/local/tomcat/webapps/ROOT
[root@centos ROOT]# ll index.html 
-rw-r--r-- 1 root root 13 Jul  2 18:57 index.html
#主页面变成index.html下的内容了

# 在ROOT/WEB-INF下将web.xml换成主页面
[root@centos WEB-INF]# ll /usr/local/tomcat/webapps/ROOT/WEB-INF
total 4
-rw-r--r-- 1 root root 531 Jul  2 19:00 web.xml


```

### 3.1 部署一个应用

```bash
#创建目录，目录可以使用的时候在创建
[root@centos ~]# cd
[root@centos ~]# mkdir projects/myapp/{WEB-INF,classes,lib} -pv
mkdir: created directory ‘projects’
mkdir: created directory ‘projects/myapp’
mkdir: created directory ‘projects/myapp/WEB-INF’
mkdir: created directory ‘projects/myapp/classes’
mkdir: created directory ‘projects/myapp/lib’

#创建应用首页页面，test.jsp
[root@centos ~]# cat projects/myapp/index.jsp
<%@ page language="java" contentType="text/html; charset=UTF-8"
 pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<head>
 <meta charset="utf-8">
 <title>jsp例子</title>
</head>
<body>
lqlixl 部署应用 test.jsp
<%
out.println("hello jsp");
%>
</body>
</html>


```



### 4.配置详情

- [root@control conf]# ll /usr/local/tomcat/conf/server.xml 

```bash

<?xml version="1.0" encoding="UTF-8"?>
<Server port="8005" shutdown="SHUTDOWN">
 <Service name="Catalina">
 <Connector port="8080" protocol="HTTP/1.1"
 connectionTimeout="20000"
 redirectPort="8443" />
 <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />
 <Engine name="Catalina" defaultHost="localhost">
 <Host name="localhost" appBase="webapps"
 unpackWARs="true" autoDeploy="true">
 </Host>
 </Engine>
 </Service>
</Server>

<GlobalNamingResources>
 <!-- Editable user database that can also be used by
 UserDatabaseRealm to authenticate users
 -->
 <Resource name="UserDatabase" auth="Container"
 type="org.apache.catalina.UserDatabase"
 description="User database that can be updated and saved"
 factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
 pathname="conf/tomcat-users.xml" />
 </GlobalNamingResources>
```

- <Server port="8005" shutdown="SHUTDOWN">
- 8005是Tomcat的管理端口，默认监听在127.0.0.1上。SHUTDOWN这个字符串接收到后就会关闭此Server

```bash
#测试
# telnet 127.0.0.1 8005
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
SHUTDOWN

#这个管理功能建议禁用，改shutdown为一串猜不出的字符串。
<Server port="8005" shutdown="44ba3c71d57f494992641b258b965f28">
```

###  5.用户认证

- 配置文件是/usr/local/tomcat/conf/tomcat-users.xml
- 打开tomcat-users.xml，我们需要一个角色manager-gui。

```bash

<tomcat-users xmlns="http://tomcat.apache.org/xml"
 xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
 xsi:schemaLocation="http://tomcat.apache.org/xml tomcat-users.xsd"
 version="1.0">
 <role rolename="manager-gui"/>
 <user username="lqlixl" password="lqlixl" roles="manager-gui"/>
</tomcat-users>
```

- Tomcat启动加载后，这些内容是常驻内存的。如果配置了新的用户，需要重启Tomcat。

### 6. 加载状态等页面错误

- 访问manager的时候告诉403，提示中告诉去manager的context.xml中修改

- 路径：/usr/local/tomcat/webapps/manager/META-INF/context.xml 

```bash
cat /usr/local/tomcat/webapps/manager/META-INF/context.xml 
<?xml version="1.0" encoding="UTF-8"?>
<Context antiResourceLocking="false" privileged="true" >
 <Valve className="org.apache.catalina.valves.RemoteAddrValve"
 allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1" />
 <Manager sessionAttributeValueClassNameFilter="java\.lang\.
(?:Boolean|Integer|Long|Number|String)|org\.apache\.catalina\.filters\.CsrfPreventionFilter\$Lru
Cache(?:\$1)?|java\.util\.(?:Linked)?HashMap"/>
</Context>

#看正则表达式就知道是本地访问了，由于当前访问地址是192.168.x.x，可以修改正则为
allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1|192\.168\.\d+\.\d+"

#在连接就成功了
```



### 7.配置虚拟主机

- 设置路径：/usr/local/tomcat/conf/server.xml

```bash
#配置文件
一般情况下，一个Server实例配置一个Service，name属性相当于该Service的ID。
<Service name="Catalina">

#连接配置
 <Connector port="8080" protocol="HTTP/1.1"
 connectionTimeout="20000"
 redirectPort="8443" />
redirectPort，如果访问HTTPS协议，自动转向这个连接器。但大多数时候，Tomcat并不会开启HTTPS，因为Tomcat往往部署在内部，HTTPS性能较差。

#引擎配置
<Engine name="Catalina" defaultHost="localhost">
defaultHost指向内部定义某虚拟主机。缺省虚拟主机可以改动，默认localhost。

#虚拟主机配置
<Host name="localhost" appBase="webapps" unpackWARs="true" autoDeploy="true">

name必须是主机名，用主机名来匹配。
appBase，当期主机的网页根目录，相对于CATALINA_HOME，也可以使用绝对路径
unpackWARs是否自动解压war格式
autoDeploy 热部署，自动加载并运行应用

```

#### 7.1 配置一个虚拟主机

- 尝试再配置一个虚拟主机，并将myapp部署到/data/webapps目录下

```bash
# 在配置文件里添加虚拟主机，并将myapp部署到/data/webapps目录下
vim /usr/local/tomcat/conf/server.xml
...
<Host name="lqlixl.com" appBase="/data/webapps/" unpackWARs="True" autoDeploy="false" />
  </Engine>
  </Service>
</Server>


#创建虚拟主机目录
mkdir /data/webapps/{WEB-INF,classes,lib} -pv

#常见应用首页，内容就用上面的test.jsp
[root@centos ~]# cat /data/webapps/myapp/index.jsp
<%@ page language="java" contentType="text/html; charset=UTF-8"
 pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<head>
 <meta charset="utf-8">
 <title>jsp例子</title>
</head>
<body>
lqlixl 部署应用 test.jsp
<%
out.println("hello jsp");
%>
</body>
</html>


#刚才在虚拟主机中主机名定义lqlixl.com，所以需要主机在本机手动配置一个域名解析如果是windows，修改在C:\Windows\System32\drivers\etc下的hosts文件，需要管理员权限。

#访问：http://lqlixl.com:8080/
```

#### 7.2 context配置

- 虚拟主机下路径映射

- 虚拟主机下，创建新的uri页面
- Context作用：
  - 路径映射
  - 应用独立配置，例如单独配置应用日志、单独配置应用访问控制

```bash
#path指的是访问的路径
#docBase，可以是绝对路径，也可以是相对路径（相对于Host的appBase）
#reloadable，true表示如果WEB-INF/classes或META-INF/lib目录下.class文件有改动，就会将WEB应用重新加载。生成环境中，会使用false来禁用。

<Context path="/test" docBase="/data/test" reloadable="" />

#创建test路径和页面
将~/projects/myapp/下面的项目文件复制到/data/下
cp -r ~/projects/myapp /data/myappv1
cd /data
ln -sv myappv1 test

#修改test.jsp区别一下

#Tomcat的配置文件server.xml中修改如下
vim /usr/local/tomcat/conf/server.xml
...
<Host name="lqlixl.com" appBase="/data/webapps/" unpackWARs="True" autoDeploy="false" >
  <Context path="test" docBase="/data/test" reloadable="" />
 </Host>
...

#重启服务

#访问页面
http://lqlixl.com:8080/test/

```

- **注意：这里特别使用了软链接，原因就是以后版本升级，需要将软链接指向myappv2，重启Tomcat。如果新版上线后，出现问题，重新修改软链接到上一个版本的目录，并重启，就可以实现回滚。**

- **注意：配置文件的标签成对出现，和“/”封口。要注意大小写。**



## 常见部署方式



![1562079736273](D:\学习资料\markdow\tomcat.assets\1562079736273.png)







- standalone模式，Tomcat单独运行，直接接受用户的请求，不推荐。
- 反向代理，单机运行，提供了一个Nginx作为反向代理，可以做到静态有nginx提供响应，动态jsp代理给
- Tomcat
  - LNMT：Linux + Nginx + MySQL + Tomcat
  - LAMT：Linux + Apache（Httpd）+ MySQL + Tomcat
- 前置一台Nginx，给多台Tomcat实例做反向代理和负载均衡调度，Tomcat上部署的纯动态页面更适合
  - LNMT：Linux + Nginx + MySQL + Tomcat

- 多级代理
  - LNNMT：Linux + Nginx + Nginx + MySQL + Tomcat







































































































































































