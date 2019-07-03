# openvpn

##  环境信息：

```bash
openvpn server：192.168.1.7    172.20.76.7
web-server1：172.20.76.17
web-server1：172.20.76.27
```

## 搭建

## 1. 服务端证书

### 1.1 安装openvpn

```bash
#安装包  epel 源
yum install openvpn easy-rsa -y

#查看包
rpm -ql easy-rsa
rpm -ql openvpn


#复制配置文件
cp /usr/share/doc/openvpn-2.4.7/sample/sample-configfiles/server.conf /etc/openvpn

cp -r /usr/share/easy-rsa/ /etc/openvpn/

cp /usr/share/doc/easy-rsa-3.0.3/vars.example
/etc/openvpn/easy-rsa/3.0.3/vars

cd /etc/openvpn/easy-rsa/3.0.3/

[root@openvpn-server 3.0.3]# tree
.
├── easyrsa
├── openssl-1.0.cnf
├── vars
└── x509-types
    ├── ca
    ├── client
    ├── COMMON
    ├── san
    └── server

```

### 1.2 创建pki和CA签发机构

```bash
[root@openvpn-server 3.0.3]# pwd
/etc/openvpn/easy-rsa/3.0.3
[root@openvpn-server 3.0.3]# ./easyrsa init-pki

Note: using Easy-RSA configuration from: ./vars

init-pki complete; you may now create a CA or requests.
Your newly created PKI dir is: /etc/openvpn/easy-rsa/3.0.3/pki

[root@openvpn-server 3.0.3]# ./easyrsa 
easyrsa          openssl-1.0.cnf  pki/             vars             x509-types/
[root@openvpn-server 3.0.3]# ll pki/
total 0
drwx------ 2 root root 6 Jun 29 14:22 private
drwx------ 2 root root 6 Jun 29 14:22 reqs

```



### 1.3 创建CA机构：

```bash

[root@openvpn-server 3.0.3]# ./easyrsa build-ca nopass

Note: using Easy-RSA configuration from: ./vars
Generating a 2048 bit RSA private key
.................+++
..............................................................+++
writing new private key to '/etc/openvpn/easy-rsa/3.0.3/pki/private/ca.key.DYA3fYdKvV'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Common Name (eg: your user, host, or server name) [Easy-RSA CA]:

CA creation complete and you may now import and sign cert requests.
Your new CA certificate file for publishing is at:
/etc/openvpn/easy-rsa/3.0.3/pki/ca.crt

[root@openvpn-server 3.0.3]# ll pki/ca.crt 
-rw------- 1 root root 1172 Jun 29 14:24 pki/ca.crt

```

###  1.4创建服务端证书(私钥)：

```bash

[root@openvpn-server 3.0.3]# ./easyrsa gen-req server nopass

Note: using Easy-RSA configuration from: ./vars
Generating a 2048 bit RSA private key
...................................+++
..................................................................+++
writing new private key to '/etc/openvpn/easy-rsa/3.0.3/pki/private/server.key.nFrvomBqMD'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Common Name (eg: your user, host, or server name) [server]:

Keypair and certificate request completed. Your files are:
req: /etc/openvpn/easy-rsa/3.0.3/pki/reqs/server.req
key: /etc/openvpn/easy-rsa/3.0.3/pki/private/server.key

# 验证CA证书
[root@openvpn-server 3.0.3]# ll ./pki/private/
total 8
-rw------- 1 root root 1704 Jun 29 14:24 ca.key
-rw------- 1 root root 1700 Jun 29 14:26 server.key
[root@openvpn-server 3.0.3]# ll ./pki/reqs/
total 4
-rw------- 1 root root 887 Jun 29 14:26 server.req

```

### 1.5 签发服务端证书

- 生成服务端crt公钥

```bash
[root@openvpn-server 3.0.3]# ./easyrsa sign server server

Note: using Easy-RSA configuration from: ./vars


You are about to sign the following certificate.
Please check over the details shown below for accuracy. Note that this request
has not been cryptographically verified. Please be sure it came from a trusted
source or that you have verified the request checksum with the sender.

Request subject, to be signed as a server certificate for 3650 days:

subject=
    commonName                = server


Type the word 'yes' to continue, or any other input to abort.
  Confirm request details: yes   #需要输入yes
Using configuration from ./openssl-1.0.cnf
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :ASN.1 12:'server'
Certificate is to be certified until Jun 26 06:28:56 2029 GMT (3650 days)

Write out database with 1 new entries
Data Base Updated

Certificate created at: /etc/openvpn/easy-rsa/3.0.3/pki/issued/server.crt

#验证生成的服务端公钥
[root@openvpn-server 3.0.3]# ll /etc/openvpn/easy-rsa/3.0.3/pki/issued/server.crt 
-rw------- 1 root root 4552 Jun 29 14:28 /etc/openvpn/easy-rsa/3.0.3/pki/issued/server.crt


```

### 1.6 ：创建 Diffie-Hellman：

- https://www.cnblogs.com/hyddd/p/7689132.html
- 密钥交换方法，由惠特菲尔德·迪菲（Bailey Whitfield Diffie）、马丁·赫尔曼（Martin Edward Hellman）于1976年发表。
- 它是一种安全协议，让双方在完全没有对方任何预先信息的条件下通过不安全信道建立起一个密钥，这个密钥一般作为“对称加密”的密钥而被双方在后续数据传输中使用。DH数学原理是base离散对数问题。做类似事情的还有非对称加密类算法，如：RSA。其应用非常广泛，在SSH、VPN、Https...都有应用，勘称现代密码基石。



```bash
[root@openvpn-server 3.0.3]# ./easyrsa gen-dh

Note: using Easy-RSA configuration from: ./vars
Generating DH parameters, 2048 bit long safe prime, generator 2
This is going to take a long time
....+.....................................................................................................................................................+.......DH parameters of size 2048 created at /etc/openvpn/easy-rsa/3.0.3/pki/dh.pem

#验证生成的秘钥文件：
[root@openvpn-server 3.0.3]# ll /etc/openvpn/easy-rsa/3.0.3/pki/dh.pem 
-rw------- 1 root root 424 Jun 29 14:34 /etc/openvpn/easy-rsa/3.0.3/pki/dh.pem

```

- 服务端证书配置完成，配置客服端证书

## 2.1 创建客服端证书

```bash
#复制客服端配置文件
[root@openvpn-server 3.0.3]# cp -r /usr/share/easy-rsa/ /etc/openvpn/client/easy-rsa
[root@openvpn-server 3.0.3]# cp /usr/share/doc/easy-rsa-3.0.3/vars.example /etc/openvpn/client/easy-rsa/vars

# 生成pki目录
[root@openvpn-server 3.0.3]# cd /etc/openvpn/client/easy-rsa/3.0.3/
[root@openvpn-server 3.0.3]# ./easyrsa init-pki

init-pki complete; you may now create a CA or requests.
Your newly created PKI dir is: /etc/openvpn/client/easy-rsa/3.0.3/pki

#验证pki目录
[root@openvpn-server 3.0.3]# ll ./pki/
total 0
drwx------ 2 root root 6 Jun 29 14:44 private
drwx------ 2 root root 6 Jun 29 14:44 reqs

[root@openvpn-server 3.0.3]# tree
.
├── easyrsa
├── openssl-1.0.cnf
├── pki  #客服端存放证书的地方
│   ├── private  
│   └── reqs
└── x509-types
    ├── ca
    ├── client
    ├── COMMON
    ├── san
    └── server


#生成客户端证书
[root@openvpn-server 3.0.3]# pwd
/etc/openvpn/client/easy-rsa/3.0.3

# 客服证书为lqlixl，没有设置密码
[root@openvpn-server 3.0.3]# ./easyrsa gen-req lqlixl nopass
Generating a 2048 bit RSA private key
...........................+++
................+++
writing new private key to '/etc/openvpn/client/easy-rsa/3.0.3/pki/private/lqlixl.key.Yf8aPtUJjX'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Common Name (eg: your user, host, or server name) [lqlixl]:

Keypair and certificate request completed. Your files are:
req: /etc/openvpn/client/easy-rsa/3.0.3/pki/reqs/lqlixl.req
key: /etc/openvpn/client/easy-rsa/3.0.3/pki/private/lqlixl.key

#查看证书
[root@openvpn-server 3.0.3]# tree /etc/openvpn/client/easy-rsa/3.0.3/pki/
/etc/openvpn/client/easy-rsa/3.0.3/pki/
├── private
│   └── lqlixl.key
└── reqs
    └── lqlixl.req

2 directories, 2 files

```

### 2.2 签发客服端证书

```bash
#切换目录
[root@openvpn-server 3.0.3]# pwd
/etc/openvpn/client/easy-rsa/3.0.3
[root@openvpn-server 3.0.3]#  cd /etc/openvpn/easy-rsa/3.0.3/
[root@openvpn-server 3.0.3]# pwd
/etc/openvpn/easy-rsa/3.0.3

#导入req文件
[root@openvpn-server 3.0.3]# ./easyrsa import-req /etc/openvpn/client/easy-rsa/3.0.3/pki/reqs/lqlixl.req  lqlixl

Note: using Easy-RSA configuration from: ./vars

The request has been successfully imported with a short name of: lqlixl
You may now use this name to perform signing operations on this request.


#签发客服端证书：
[root@openvpn-server 3.0.3]# ./easyrsa sign client lqlixl

Note: using Easy-RSA configuration from: ./vars


You are about to sign the following certificate.
Please check over the details shown below for accuracy. Note that this request
has not been cryptographically verified. Please be sure it came from a trusted
source or that you have verified the request checksum with the sender.

Request subject, to be signed as a client certificate for 3650 days:

subject=
    commonName                = lqlixl


Type the word 'yes' to continue, or any other input to abort.
  Confirm request details: yes 
Using configuration from ./openssl-1.0.cnf
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :ASN.1 12:'lqlixl'
Certificate is to be certified until Jun 26 07:01:29 2029 GMT (3650 days)

Write out database with 1 new entries
Data Base Updated

Certificate created at: /etc/openvpn/easy-rsa/3.0.3/pki/issued/lqlixl.crt

# 验证签发后的crt证书
[root@openvpn-server 3.0.3]# ll /etc/openvpn/easy-rsa/3.0.3/pki/issued/lqlixl.crt 
-rw------- 1 root root 4432 Jun 29 15:01 /etc/openvpn/easy-rsa/3.0.3/pki/issued/lqlixl.crt
```

### 2.3 复制证书到server目录

```bash 
#创建目录，并进入
[root@openvpn-server 3.0.3]# mkdir /etc/openvpn/certs
[root@openvpn-server 3.0.3]# cd /etc/openvpn/certs

#复制证书
[root@openvpn-server certs]# cp /etc/openvpn/easy-rsa/3.0.3/pki/dh.pem .
[root@openvpn-server certs]# cp /etc/openvpn/easy-rsa/3.0.3/pki/issued/server.crt .
[root@openvpn-server certs]# cp /etc/openvpn/easy-rsa/3.0.3/pki/private/server.key .
[root@openvpn-server certs]# cp /etc/openvpn/easy-rsa/3.0.3/pki/ca.crt .
[root@openvpn-server certs]# tree
.
├── ca.crt
├── dh.pem
├── server.crt
└── server.key

0 directories, 4 files

```

### 2.4 客服端公钥和私钥

```bash
[root@openvpn-server certs]# mkdir /etc/openvpn/client/lqlixl/
[root@openvpn-server certs]# cp /etc/openvpn/easy-rsa/3.0.3/pki/ca.crt  /etc/openvpn/client/lqlixl/
[root@openvpn-server certs]# cp /etc/openvpn/easy-rsa/3.0.3/pki/issued/lqlixl.crt /etc/openvpn/client/lqlixl/
[root@openvpn-server certs]# cp /etc/openvpn/client/easy-rsa/3.0.3/pki/private/lqlixl.key /etc/openvpn/client/lqlixl/

# 查看目录文件
[root@openvpn-server certs]# tree /etc/openvpn/client/lqlixl/
/etc/openvpn/client/lqlixl/
├── ca.crt
├── lqlixl.crt
└── lqlixl.key

0 directories, 3 files
[root@openvpn-server certs]# 



```



### 2.5 server端配置文件

```bash
[root@openvpn-server ~]# vim /etc/openvpn/server.conf 
local 172.18.200.102 #本机监听IP
port 1194 #端口
# TCP or UDP server?
proto tcp #协议，指定OpenVPN创建的通信隧道类型
#proto udp
#dev tap：创建一个以太网隧道，以太网使用tap
dev tun：创建一个路由IP隧道，互联网使用tun
一个TUN设备大多时候，被用于基于IP协议的通讯。一个TAP设备允许完整的以太网帧通过Openvpn隧道，因此
提供非ip协议的支持，比如IPX协议和AppleTalk协议
A TUN device is used mostly for VPN tunnels where only IP-traffic is used. A TAP
device allows full Ethernet frames to be passed over the OpenVPN tunnel. hence
providing support for non-ip based protocols such as IPX and AppleTalk.
#dev-node MyTap #TAP-Win32适配器。非windows不需要
#topology subnet #网络拓扑，不需要配置
server 10.8.0.0 255.255.255.0 #客户端连接后分配IP的地址池，服务器默认会占用第一个IP
10.8.0.1
#ifconfig-pool-persist ipp.txt #为客户端分配固定IP，不需要配置
#server-bridge 10.8.0.4 255.255.255.0 10.8.0.50 10.8.0.100 #配置网桥模式，不需要
push " route 10.20.0.0 255.255.0.0" #给客户端生成的静态路由表，下一跳为openvpn服务器的
10.8.0.1
push " route 172.31.0.0 255.255.16.0"
;client-config-dir ccd #为指定的客户端添加路由，改路由通常是客户端后面的内网网段而不是服务端
的，也不需要设置
;route 192.168.40.128 255.255.255.248
;client-config-dir ccd
;route 10.9.0.0 255.255.255.252
;learn-address ./script #运行外部脚本，创建不同组的iptables 规则，不配置
;push "redirect-gateway def1 bypass-dhcp" #启用后，客户端所有流量都将通过VPN服务器，因此不
需要配置
#;push "dhcp-option DNS 208.67.222.222" #推送DNS服务器，不需要配置
#;push "dhcp-option DNS 208.67.220.220"
client-to-client #运行不同的client直接通信
;duplicate-cn #多个用户共用一个账户，一般用于测试环境，生产环境都是一个用户一个证书
keepalive 10 120 #设置服务端检测的间隔和超时时间，默认为每 10 秒 ping一次，如果 120 秒没有回
应则认为对方已经 down
#tls-auth /etc/openvpn/server/ta.key 0 #可使用以下命令来生成：openvpn –genkey –secret
ta.key #服务器和每个客户端都需要拥有该密钥的一个拷贝。第二个参数在服务器端应该为’0’，在客户端应
该为’1’
cipher AES-256-CBC #加密算法
;compress lz4-v2 #启用压缩
;push "compress lz4-v2"
;comp-lzo #旧户端兼容的压缩配置，需要客户端配置开启压缩
;max-clients 100 #最大客户端数
;user nobody #运行openvpn服务的用户和组
;group nobody
persist-key 久化选项可以尽量避免访问那些在重启之后由于用户权限降低而无法访问的某些资源
persist-tun
status openvpn-status.log #openVPN状态记录文件，每分钟会记录一次
#;log openvpn.log #日志记录方式和路径，log会在openvpn启动的时候清空日志文件
log-append /var/log/openvpn/openvpn.log #重启openvpn后在之前的日志后面追加新的日志
verb 3 #设置日志级别，0-9，级别越高记录的内容越详细，
mute 20 #相同类别的信息只有前20条会输出到日志文件中
explicit-exit-notify 1 #通知客户端，在服务端重启后可以自动重新连接，仅能用于udp模式
```

#### 2.5.1 创建日志文件

```bash
[root@openvpn-server ~]# mkdir /var/log/openvpn
#授权
[root@openvpn-server ~]# chown nobody.nobody /var/log/openvpn
```

#### 2.5.2 最终配置

```bash
[root@openvpn-server ~]# grep "^[a-Z]" /etc/openvpn/server.conf
local 172.20.76.7 
port 1194
proto tcp
dev tun
ca /etc/openvpn/certs/ca.crt
cert /etc/openvpn/certs/server.crt
key /etc/openvpn/certs/server.key  # This file should be kept secret
dh /etc/openvpn/certs/dh.pem
server 192.168.1.0 255.255.255.0
ifconfig-pool-persist ipp.txt
push "route 192.168.1.0 255.255.255.0"
push "route 10.20.0.0 255.255.0.0"
client-to-client
keepalive 10 120
cipher AES-256-CBC
max-clients 100
user nobody
group nobody
persist-key
persist-tun
status openvpn-status.log
log-append  /var/log/openvpn/openvpn.log
verb 9
mute 20


```

###  2.6 客服端配置文件

```bash
#复制客服端的配置文件
[root@openvpn-server lqlixl]# cp /usr/share/doc/openvpn-2.4.7/sample/sample-config-files/client.conf /etc/openvpn/client/lqlixl/client.ovpn

#最总客服端配置内容
[root@openvpn-server ~]# cd /etc/openvpn/client/lqlixl/
[root@openvpn-server lqlixl]#  grep -Ev "^(#|$|;)" /etc/openvpn/client/lqlixl/client.ovpn
client
dev tun
proto tcp
remote 192.168.1.17 1194
resolv-retry infinite
nobind
persist-key
persist-tun
ca ca.crt
cert lqlixl.crt
key lqlixl.key
remote-cert-tls server
tls-auth ta.key 1
cipher AES-256-CBC
verb 3

#验证当前目录
[root@openvpn-server lqlixl]# tree /etc/openvpn/client/lqlixl/
/etc/openvpn/client/lqlixl/
├── ca.crt
├── client.ovpn
├── lqlixl.crt
└── lqlixl.key

0 directories, 4 files
```

## 3 启动openvpn服务：

### 3.1 启动前配置

```bash
#禁用firewalld
[root@openvpn-server lqlixl]# systemctl stop firewalld
[root@openvpn-server lqlixl]# systemctl disable firewalld

#安装iptables
[root@openvpn-server lqlixl]# yum install iptables-services.x86_64 iptables -y

#启动服务和开机自启
[root@openvpn-server lqlixl]# systemctl enable iptables.service 
Created symlink from /etc/systemd/system/basic.target.wants/iptables.service to /usr/lib/systemd/system/iptables.service.
[root@openvpn-server lqlixl]# systemctl start iptables.service 

#清空已有的防火墙规则
[root@openvpn-server lqlixl]# iptables -F

#开启路由转发
[root@openvpn-server lqlixl]# sysctl -p
net.ipv4.ip_forward = 1

#创建iptables 规则
[root@openvpn-server lqlixl]# iptables -t nat -A POSTROUTING -s 10.20.0.0/16 -j ACCEPT
[root@openvpn-server lqlixl]# iptables -A INPUT -p TCP --dport 1194 -j ACCEPT
[root@openvpn-server lqlixl]# iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

#保存规则并查看规则
[root@openvpn-server lqlixl]# service iptables save
iptables: Saving firewall rules to /etc/sysconfig/iptables:[  OK  ]

[root@openvpn-server lqlixl]# iptables -vnL
Chain INPUT (policy ACCEPT 8 packets, 1832 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:1194
   34  2244 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 22 packets, 2160 bytes)
 pkts bytes target     prot opt in     out     source               destination 
 
 #查看你nat
 [root@openvpn-server lqlixl]# iptables -t nat -vnL
Chain PREROUTING (policy ACCEPT 8 packets, 1832 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain INPUT (policy ACCEPT 8 packets, 1832 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 ACCEPT     all  --  *      *       10.20.0.0/16         0.0.0.0/0  

```



### 3.2 启动服务

```bash
systemctl start openvpn@server
systemctl enable openvpn@server

#验证日志
tail /var/log/openvpn/openvpn.log

#验证tun网卡设备：
ifcofing
```

### 3.3 将文件打包传到桌面

```bash
[root@openvpn-server ~]# cd /etc/openvpn/client/lqlixl/
[root@openvpn-server zhangshijie]# tar czvf lqlixl.tar.gz ./*
[root@openvpn-server lqlixl]# ls
ca.crt  client.ovpn  lqlixl.crt  lqlixl.key  lqlixl.tar.gz
[root@openvpn-server lqlixl]# sz lqlixl.tar.gz 



#放到安装的软件config里面
OpenVPN\config
```

### 3.4 软件

```bash
官方客户端下载地址： https://openvpn.net/community-downloads/
非官方地址：
https://sourceforge.net/projects/securepoint/files/

```















































































































































































































































































































