# docker

## 安装使用docker

### 1. 使用阿里云安装




```bash
#配置安装源
vim /etc/apt/sources.list
...
https://opsx.alibaba.com/mirror
...
#更新
apt-get update
apt upgraded

#安装相关的系统工具
https://yq.aliyun.com/articles/110806
apt-get -y install apt-transport-https ca-certificates curl software-properties-common
#安装gpg证书
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
#写入软件源信息
sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"

#安装并更新docker-ce
apt-get -y update
apt-get -y install docker-ce

```

### 2. 启动使用

```bash
#启动docker并开机启动
systemctl start docker
systemctl enable docker
systemctl status docker

#验证 docker0 网卡：
在 docker 安装启动之后，默认会生成一个名称为 docker0 的网卡并且默认 IP 地址为 172.17.0.1 的网卡。

使用ifconfig  查看

#查看dockers版本信息
docker version

#查看所有信息
docker info


```



### 3. 查看 containerd 进程关系：
- 有四个进程：
  dockerd：被 client 直接访问，其父进程为宿主机的 systemd 守护进程。
  docker-proxy：实现容器通信，其父进程为 dockerd
  containerd：被 dockerd 进程调用以实现与 runc 交互。
  containerd-shim：真正运行容器的载体，其父进程为 containerd。

- dockerd   调用containerd 

  containerd 调用containerd-shim 生成容器

  containerd 返回dockerd  

  dockerd 通过proxy 生成防火规则等

  

### 4.镜像管理

#### 4.1  建立阿里云镜像加速

```bash
#来源
https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors

#配置
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://0gemcyn8.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker


#拉取centos镜像到本地
docker pull centos

重启 docke 服务：
systemctl daemon-reload
sudo systemctl restart docker
```

![1562641326001](D:\学习资料\markdow\Basic\docker\docker.assets\1562641326001.png)

#### 4.2  搜索镜像

```bash
#在官方的 docker 仓库中搜索指定名称的 docker 镜像，也会有很多三方镜像。
docker search centos:7.2.1511 #带指定版本号
docker search centos #不带版本号默认 latest

# tags
区分版本环境的
```

#### 4.3 下载镜像

```bash
#从 docker 仓库将镜像下载到本地，命令格式如下：
docker pull 仓库服务器:端口/项目名称/镜像名称:tag(版本)号

docker pull alpine
docker pull nginx
docker pull hello-world
docker pull centos

#查看本地的镜像
下载完成的镜像比下载的大，因为下载完成后会解压
root@ubuntu-1804:~# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED         SIZE
nginx               latest              f68d6e55e065        7 days ago     109MB
alpine              latest              4d90542f0623        2 weeks ago    5.58MB
centos              latest              9f38484d220f        3 months ag    202MB
hello-world         latest              fce289e99eb9        6 months ago    1.84kB

REPOSITORY #镜像所属的仓库名称
TAG #镜像版本号（标识符），默认为 latest
IMAGE ID #镜像唯一 ID 标示
CREATED #镜像创建时间
VIRTUAL SIZE #镜像的大小

```

#### 4.4 镜像导出

```bash
#可以将镜像从本地导出为一个压缩文件，然后复制到其他服务器进行导入使用。
#导出方法 1：
[root@docker-server1 ~]# docker save centos -o /opt/centos.tar.gz
[root@docker-server1 ~]# ll /opt/centos.tar.gz
 /opt/centos.tar.gz
#导出方法 2：
[root@docker-server1 ~]# docker save centos > /opt/centos-1.tar.gz
[root@docker-server1 ~]# ll /opt/centos-1.tar.gz
 /opt/centos-1.tar.gz

```

#### 4.5 镜像导入

```bash
#将镜像导入到 docker
[root@docker-server1 ~]# scp /opt/centos.tar.gz 192.168.10.206:/opt/
[root@docker-server2 ~]# docker load < /opt/centos.tar.gz

```

#### 4.6 删除镜像

```bash
[root@docker-server1 opt]# docker rmi centos

#获取运行参数帮助
[root@linux-docker opt]# docker daemon –help
总结：企业使用镜像及常见操作：
搜索、下载、导出、导入、删除
```

#### 4.7 总结

```bash
命令总结：
# docker load -i centos-latest.tar.xz #导入本地镜像
# docker save > /opt/centos.tar #centos #导出镜像
# docker rmi 镜像 ID/镜像名称 #删除指定 ID 的镜像，通过镜像启动容器的时候镜像不能被删除，除非将容器全部关闭
# docker rm 容器 ID/容器名称 #删除容器
# docker rm 容器 ID/容器名-f #强制删除正在运行的容器
```

### 5.容器操作基础命令


```bash
#命令格式：
# docker run [选项] [镜像名] [shell 命令] [参数]
# docker run [参数选项] [镜像名称，必须在所有选项的后面] [/bin/echo 'hello wold'] #单次执行，没有自定义容器名称
# docker run centos /bin/echo 'hello wold' #启动的容器在执行完 shel 命令就退出了

```

#### 5.1 从镜像启动一个容器

- 会直接进入到容器，并随机生成容器 ID 和名称

```bash
root@ubuntu-1804:~# docker run -it centos bash
[root@e3309edebbe0 /]# ls
anaconda-post.log  dev  home  lib64  mnt  proc  run   srv  tmp  var
 bin   etc  lib   media  opt  root  sbin  sys  usr

#退出容器不注销
ctrl+p+q

# -i -t 创建然后进入容器

：/var/lib/containerd/io.containerd.runtime.v1.linux/moby/容器 ID
```

#### 5.2 显示容器

```bash
#显示正在运行的容器：
root@ubuntu-1804:~# docker ps


#显示所有容器：
包括当前正在运行以及已经关闭的所有容器：
root@ubuntu-1804:~# docker ps -a

```

#### 5.3 删除运行中的容器

```bash
#即使容器正在运行当中，也会被强制删除掉
root@ubuntu-1804:~# docker rm -f e3309edebbe0

#查看
root@ubuntu-1804:~# docker ps -a
```

#### 5.4 端口映射

```bash
#随机映射端口
	root@ubuntu-1804:~# docker pull nginx
	root@ubuntu-1804:~#docker run -P docker.io/nginx #前台启动并随机映射本地端口到容器的 80
	
	#前台启动的会话窗口无法进行其他操作，除非退出，但是退出后容器也会退出
	#随机端口映射，其实是默认从 32768 开始
	
	#查看容器
	docker ps -a
	docker exec -it 936b5bec0415 bash
	ss -ntl
	0.0.0.0:32768->80/tcp
	
	#浏览器访问
	http://192.168.1.150:32768/


#指定端口映射：
	方式 1：本地端口 81 映射到容器 80 端口：
	# docker run -p 81:80 --name nginx-test1-port1 nginx
	方式 2：本地 IP:本地端口:容器端口
	# docker run -p 192.168.10.205:82:80 --name nginx-test-port2 docker.io/nginx
	方式 3：本地 IP:本地随机端口:容器端口
	# docker run -p 192.168.10.205::80 --name nginx-test-port3 docker.io/nginx
	方式 4：本机 ip:本地端口:容器端口/协议，默认为 tcp 协议
	# docker run -p 192.168.10.205:83:80/udp --name nginx-test-port4 docker.io/nginx
	方式 5：一次性映射多个端口+协议：
	# docker run -p 86:80/tcp -p 443:443/tcp -p 53:53/udp --name nginx-test-port5 docker.io/nginx
	
	
#查看运行的容器：
docker ps

#查看nginx 容器访问日志
docker logs nginx-test2-port2 #一次查看
docker logs -f nginx-tes2-port2 #持续查看

#查看容器已经映射的端口
root@ubuntu-1804:~# docker port nginx-test2-port2
80/tcp -> 0.0.0.0:82

```

#### 5.6 自定义容器的名称

```bash
#命名为nginx-test
docker run -it --name nginx-test nginx

```

#### 5.7后台启动容器

```bash
root@ubuntu-1804:~# docker run -d -P --name nginx-test1 nginx
a1937c4e6e2ab2a3194ec78b250d831cf795de5e0057dec65a2f8d15e56b2edd


```

#### 5.8创建并进入容器

```bash
docker run -it root@ubuntu-1804:~# docker run -t -i --name test-centos centos bash
[root@5b9579575cea /]# ps -aux
USER        PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root          1  0.3  0.1  11832  2944 pts/0    Ss   08:34   0:00 bash
root         14  0.0  0.1  51752  3416 pts/0    R+   08:34   0:00 ps -aux--name test-centos2 docker.io/centos /bin/bash

#创建容器后直接进入，执行 exit 退出后容器关闭
```

#### 5.9 单次运行

- 容器退出后自动删除

```bash
root@ubuntu-1804:~# docker run -it --rm --name nginx-delete-test nginx

#用于测试代码上线等

```

####  5.10传递运行命令

- 容器需要有一个前台运行的进程才能保持容器的运行，通过传递运行参数是一种方式，另外也可以在构建镜像的时候指定容器启动时运行的前台命令。

```bash
root@ubuntu-1804:~# docker run -d centos /usr/bin/tail -f '/etc/hosts'
365ee9d0deddec552ca338955bf65cb593a674d470002c68ec96a252ee690df9
root@ubuntu-1804:~# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
365ee9d0dedd        centos              "/usr/bin/tail -f /e…"   19 seconds ago      Up 18 seconds      flamboyant_wozniak


```

#### 5.11 容器的开启和关闭

```bash
#查看容器ID
docker ps
#关闭容器
docker stop f821d0cd5a99
#开启容器
docker start f821d0cd5a99
```

####  5.12 查看容器内部的 hosts 文件

```bash
root@ubuntu-1804:~#  docker run -i -t --name test-centos3 docker.io/centos /bin/bash

[root@d3dc733ad341 /]# cat /etc/hosts
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
172.17.0.3	d3dc733ad341  #默认会将实例的 ID 添加到自己的 hosts 文件

#ping 容器 ID：
[root@d3dc733ad341 /]# ping d3dc733ad341
PING d3dc733ad341 (172.17.0.3) 56(84) bytes of data.
64 bytes from d3dc733ad341 (172.17.0.3): icmp_seq=1 ttl=64 time=0.035 ms

```

#### 5.13 删除容器

```bash
#强制关闭所有运行中的容器
 docker kill $(docker ps -a -q) 
docker kill `docker ps -a -q`
#批量删除已退出容器：
 docker rm -f `docker ps -aq -f status=exited`
 
#批量删除所有容器：
 docker rm -f $(docker ps -a -q)
  docker rm -f `docker ps -a -q`
```

####  5.14 指定容器的DNS

-  Dns 服务，默认采用宿主机的 dns 地址
  - 一是将 dns 地址配置在宿主机
  - 二是将参数配置在 docker 启动脚本里面 –dns=1.1.1.1

```bash
[root@docker-server1 ~]# docker run -it --rm --dns 223.6.6.6 centos bash
[root@afeb628bf074 /]# cat /etc/resolv.conf
nameserver 223.6.6.6

```

##  Docker 镜像制作

### 1. 手动制作 yum 版 nginx 镜像

- Docker 制作类似于虚拟机的镜像制作，即按照公司的实际业务务求将需要安装的软件、相关配置等基础环境配置完成，然后将其做成镜像，最后再批量从镜像批量生产实例，这样可以极大的简化相同环境的部署工作，Docker 的镜像制作分为手动制作和自动制作(基于 DockerFile)，其中手动制作镜像步骤具体如下：
- 基于某个基础镜像之上重新制作，因此需要先有一个基础镜像，本次使用官方提供的 centos 镜像为基础：

```bash
[root@docker-server1 ~]# docker pull centos
[root@docker-server1 ~]# docker run -it docker.io/centos /bin/bash
[root@37220e5c8410 /]# yum install wget -y
[root@37220e5c8410 /]# cd /etc/yum.repos.d/
[root@37220e5c8410 yum.repos.d]# rm -rf ./* #更改 yum 源
#在阿里云镜像源首页复制出来
[root@37220e5c8410 yum.repos.d]# wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
[root@37220e5c8410 yum.repos.d]# wget -O /etc/yum.repos.d/epel.repo  http://mirrors.aliyun.com/repo/epel-7.repo

```

#### 1.2. yum 安装并配置 nginx：

```bash
[root@37220e5c8410 yum.repos.d]# yum install nginx –y #yum 安装 nginx
[root@37220e5c8410 yum.repos.d]# yum install -y vim wget pcre pcre-devel zlib zlib-devel openssl openssl-devel iproute net-tools iotop #安装常用命令
```

#### 1.3 关闭nginx 后台执行

```bash
[root@37220e5c8410 yum.repos.d]# vim /etc/nginx/nginx.conf #关闭 nginx 后台运
行
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;
daemon off; #关闭后台运行

```

#### 1.4. 自定义 web 页面：

```bash
[root@37220e5c8410 yum.repos.d]# vim /usr/share/nginx/html/index.html
[root@37220e5c8410 yum.repos.d]# cat /usr/share/nginx/html/index.html
Docker Yum Nginx #自定义 web 界面

```

#### 1.5. 提交为镜像：

```bash
在宿主机基于容器 ID 提交为镜像
[root@docker-server1 ~]# docker commit -m "nginx image" f5f8c13d0f9f jack/centos-nginx /usr/sbin/nginx
sha256:7680544414220f6ff01e58ffc6bd59ff626af5dbbce4914ceb18e20f4ae0276a

```

#### 1.6. 带 tag 的镜像提交：

```bash
提交的时候标记 tag 号：
#标记 tag 号，生产当中比较长用，后期可以根据 tag 标记启动不同版本启动 image启动
root@ubuntu-1804:~# docker commit -m "nginx image" de2a1e9129fd jack/centos-nginx:v1
sha256:aa7cec225579fd64235edfbc883c993670c482efecc559a6d4c58159ea3ad3f5

```

#### 1.7.从自己镜像启动容器：

```bash
root@ubuntu-1804:~# docker run -d -p 80:80 --name my-centos-nginx jack/centos-nginx /usr/sbin/nginx
72b4003ec57c7ea69c2c883752ca3f0e9fb64cda2faf68352a43c0209f9e4af9
```

#### 1.8 访问测试：

```bash
http://192.168.1.150/
```

## DockerFile 制作 yum 版 nginx 镜像： 

- DockerFile 可以说是一种可以被 Docker 程序解释的脚本，DockerFile 是由一条条的命令组成的，每条命令对应 linux 下面的一条命令，Docker 程序将这些DockerFile 指令再翻译成真正的 linux 命令，其有自己的书写方式和支持的命令，Docker 程序读取 DockerFile 并根据指令生成 Docker 镜像，相比手动制作镜像的方式，DockerFile 更能直观的展示镜像是怎么产生的，有了 DockerFile，当后期有额外的需求时，只要在之前的 DockerFile 添加或者修改响应的命令即可重新生成新的 Docke 镜像，避免了重复手动制作镜像的麻烦，具体如下：



### 1 下载镜像并初始化系统：

```bash
[root@docker-server1 ~]# docker pull centos
[root@docker-server1 ~]# docker run -it docker.io/centos /bin/bash
[root@docker-server1 ~]# cd /opt/ #创建目录环境
[root@docker-server1 opt]# mkdir dockerfile/{web/{nginx,tomcat,jdk,apache},system/{centos,ubuntu,redhat}} -pv
#目录结构按照业务类型或系统类型等方式划分，方便后期镜像比较多的时候进行分类。

[root@docker-server1 opt]# cd dockerfile/web/nginx/
[root@docker-server1 nginx]# pwd
/opt/dockerfile/web/nginx
```

### 2. 编写 Dockerfile：

```bash
[root@docker-server1 nginx]# vim ./Dockerfile #生成的镜像的时候会在执行命令的当前目录查找 Dockerfile 文件，所以名称不可写错，而且 D 必须大写
#My Dockerfile
#"#"为注释，等于 shell 脚本的中#
#除了注释行之外的第一行，必须是 From xxx (xxx 是基础镜像)
From centos #第一行先定义基础镜像，后面的本地有效的镜像名，如果本地没
有会从远程仓库下载，第一行很重要
#镜像维护者的信息
MAINTAINER Jack.Zhang 123456@qq.com

# 其 他 可 选 参 数
###########################
#USER #指定该容器运行时的用户名和 UID，后续的 RUN 命令也会使用这面指定的用户执行
#WORKDIR /a
#WORKDIR b #指定工作目录，最终为/a/b
#VOLUME ["/dir_1", "/dir_2" ..] 设置容器挂载主机目录
#ENV name jack #设置容器变量，常用于想容器内传递用户密码等
#########################


#执行的命令，将编译安装 nginx 的步骤执行一遍
RUN rpm -ivh http://mirrors.aliyun.com/epel/epel-release-latest-7.noarch.rpm
RUN yum install -y vim wget tree lrzsz gcc gcc-c++ automake pcre pcre-devel zlib zlibdevel openssl openssl-devel iproute net-tools iotop
ADD nginx-1.10.3.tar.gz /usr/local/src/ #自动解压压缩包
RUN cd /usr/local/src/nginx-1.10.3 && ./configure --prefix=/usr/local/nginx --
with-http_sub_module && make && make install
RUN cd /usr/local/nginx/
ADD nginx.conf /usr/local/nginx/conf/nginx.conf
RUN useradd nginx -s /sbin/nologin
RUN ln -sv /usr/local/nginx/sbin/nginx /usr/sbin/nginx
RUN echo "test nginx page" > /usr/local/nginx/html/index.html

EXPOSE 80 443 #向外开放的端口，多个端口用空格做间隔，启动容器时候-p 需
要使用此端向外映射，如： -p 8081:80，则 80 就是这里的 80
CMD ["nginx"] #运行的命令，每个 Dockerfile 只能有一条，如果有多条则只有最后
一条被执行
#如果在从该镜像启动容器的时候也指定了命令，那么指定的命令会覆盖
Dockerfile 构建的镜像里面的 CMD 命令，即指定的命令优先级更高，Dockerfile 的
优先级较低一些

```

### 访问仓库

- Nexus





1. 求解   $$





