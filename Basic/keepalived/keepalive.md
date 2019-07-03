## keepalive

### 1.高可用集群概念

#### 1.1 集群类型

- LB  lvs/nginx(http/upstream,stream/upstream)
- HA 高可用性
- HPC高可用性集群

#### 1.2系统可用性：SLA（Service-Level-Agreement）

- 95%=(60*24*30)*(1-0.9995)
  - （指标）=99%, ..., 99.999%，99.9999%

#### 1.3系统故障：

- 硬件故障：设计缺陷、wear out（损耗）、自然灾害……
- 软件故障：设计缺陷

#### 1.5提升系统高可用性降低MTTR（平均故障时间）

- 解决方案：建立冗余机制
  - active/passive 主/备
    active/active 双主
    active --> HEARTBEAT --> passive
    active <--> HEARTBEAT <--> active

#### 1.6双节点集群

- 辅助设备：ping node, quorum disk(仲裁设备)
  • Failover：故障切换，即某资源的主节点故障时，将资源转移至其它节点的操作
  • Failback：故障移回，即某资源的主节点故障后重新修改上线后，将之前已转移至其
  它节点的资源重新切回的过程

### 2.keepalicve简介

#### 2.1keepalived:

- vrrp协议的软件实现，原生设计目的为了高可用ipvs服务

#### 2.2功能

- • 基于vrrp协议完成地址流动
  • 为vip地址所在的节点生成ipvs规则(在配置文件中预先定义)
  • 为ipvs集群的各RS做健康状态检测
  • 基于脚本调用接口通过执行脚本完成脚本中定义的功能，进而影响集群事务，以此
  支持nginx、haproxy等服务

#### 2.3术语

- 虚拟路由器：Virtual Router
  虚拟路由器标识：VRID(0-255)，唯一标识虚拟路由器
  物理路由器：
  	master：主设备
  	backup：备用设备
  	priority：优先级

- VIP：Virtual IP
  VMAC：Virutal MAC (00-00-5e-00-01-VRID)

#### 2.3通告：

-  心跳，优先级等；周期性

#### 2.4工作方式

- 抢占式，修好后，将业务在此抢占回来
- 非抢占式。修好不管前面的业务。

#### 2.5工作模式：

- 主/备：单虚拟路由器
- 主/主：主/备（虚拟路由器1），备/主（虚拟路由器2）
  - 主主就是互为主备。两个VIP；公司一般将一个网段的地址250个分开。每个高可用性上跑125个

#### 2.6脑裂

- A和B两台机器，A认为**B**出问题，B认为A出问题。A,B都接管服务称之为脑裂
- 解决办法
  - 找一根网线连接A和B连接一根网线（网线别断）
  - 第三方仲裁者

### 3.keepalived搭建安装

#### 3.1环境准备

- 各节点时间必须同步
- 关闭selinux和防火墙

#### 3.2keepalived安装

- CentOS: yum install keepalived -y 
- Ubuntu: apt-get install keepalived 

#### 3.3程序环境：

- 查看通用的编译参数 keepalived -v
- 查看安装的各类文件：rpm -ql keepalived

- 主配置文件：/etc/keepalived/keepalived.conf
- 主程序文件：/usr/sbin/keepalived
- Unit File（启动脚本）：
  - CentOS: /usr/lib/systemd/system/keepalived.service
  - Ubuntu:  /lib/systemd/system/keepalived.service

- Unit File的环境配置文件，配置的会传递给启动脚本：
  - /etc/sysconfig/keepalived

- 启动服务,开机启动
  - systemctl start keepalived
  - systemctl enable keepalived

#### 3.4安装辅助工具

- ipvsadm是LVS在应用层的管理命令，我们可以通过这个命令去管理查看LVS配置的选项。
- lvs基于内核的。

- yum install ipvsadm.x86_64 -y
- 使用命令查看生成的规则
  - ipvsadm -Ln 
  - ipvsadm -Ln --stats

### 4.配置主配置文件

#### 4.1使用组播设置主备路由器

```bash
#组播配置的MASTER:   172.20.76.28
vim /etc/keepalived/keepalived.conf 
global_defs  {    #全局配置        
   notification_email {  #指定keepalived在发生切换时需要发送email到的对象，可以放多个，一行一个
    root@xiangzi.net
   }
   notification_email_from root@xiangzi.net #指定发件人
   smtp_server localhost #指定smtp服务器地址，stmp:简单的邮件传输协议
   smtp_connect_timeout 30 #指定smtp连接超时时间
   #不严格遵循vrrp 规则不然后面起不来，进程相关的。优化参数
   router_id LVS_DEVEL #运行keepalived机器的一个标识,主机名保证每一个都不一样
   vrrp_skip_check_adv_addr #跳过检查数据报文，打开消耗性能
   vrrp_strict #严格遵守VRRP协议,不允许状况:1,没有VIP地址,2.单播邻居,3.在VRRP版本2中有IPv6地址.
   vrrp_garp_interval 0     #ARP报文发送延迟
   vrrp_gna_interval 0    #消息发送延迟
   vrrp_mcast_group4 224.0.0.18 #默认组播IP地址，224.0.0.0到239.255.255.255
   vrrp_iptables   #原来有iptables规则，拒绝所有访问的包，可以访问出去，不能回来；打开可以关闭iptables规则;

}

vrrp_instance VIP1 {      #设置实例名称一般使用业务相关的名称  VIP1    
    state MASTER      # 描述    
    interface eth0   #生成的网络接口，要网络接口要存在。
    virtual_router_id 76    #虚拟的路由ID，0~255，不要与人冲突。
    priority 100    #优先级，通常master优先级比slave高50
    advert_int 2     #设置默认探测时间，1s太短，网络有时候就需要2，3秒。
    authentication {    #认证机制
        auth_type PASS
        auth_pass redhat   #仅前八位数有效
    }   
    virtual_ipaddress {    #虚拟的VIP地址，可以写多个VIP
        172.20.76.100 dev eth0 label eth0:0   #绑定的接口是eth0，标签子接口是 eth0:0      
    }   
}
#重启服务
systemctl restart keepalived.service 

#组播配置的BACKUP:   172.20.76.28
vim /etc/keepalived/keepalived.conf 
 router_id slave
 vrrp_iptables 
vrrp_instance VIP1 {
    state BACKUP        
    interface eth0
    virtual_router_id 76   
    priority 80
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass redhat
    }   
    virtual_ipaddress {
        172.20.76.100 dev eth0 label eth0:0                                          
    }   
}
#重启服务
systemctl restart keepalived.service


#在主从上都要设置：master:172.20.76.18   slave:172.20.76.28
sysctl -a   #查看所有内核参数
#监听一个不存在的IP需要添加设置，表示是否允许进程邦定到非本地IP地址。
sysctl -a |grep local
net.ipv4.ip_nonlocal_bind = 0 #将0修改成1
#Linux系统没有打开IP转发功能，要确认IP转发功能的状态，可以查看/proc文件系统/proc/sys/net/ipv4/ip_forward，或者下面命令：
sysctl -a  |grep ip_forward  #查找ip_forward
net.ipv4.ip_forward = 0  #永久修改需要将0需改成1
#将刚刚修改的文件生效
sysctl -p
net.ipv4.ip_nonlocal_bind = 1
net.ipv4.ip_forward = 1
#永久修改网络参数，需要修改/etc/sysctl.conf文件，修改下面一行的值：
cat /etc/sysctl.conf 
net.ipv4.ip_nonlocal_bind = 1
net.ipv4.ip_forward = 1 


#设置haproxy 172.20.76.18
vim /etc/haproxy/haproxy.cfg 
listen WEB_PORT_80 
    bind 172.20.76.187:80                                                                
    mode http
    balance roundrobin
    server 172.20.76.38 172.20.76.38:80  weight 1 check inter 3s fall 3 rise 5
    server 172.20.76.48 172.20.76.48:80  weight 1 check inter 3s fall 3 rise 5
#重启服务
systemctl restart haproxy.service

#设置haproxy 172.20.76.28
vim /etc/haproxy/haproxy.cfg 	
listen WEB_PORT_80 
    bind 172.20.76.187:80                                                                
    mode http
    balance roundrobin
    server 172.20.76.38 172.20.76.38:80  weight 1 check inter 3s fall 3 rise 5
    server 172.20.76.48 172.20.76.48:80  weight 1 check inter 3s fall 3 rise 5
#重启服务
systemctl restart haproxy.service

#通过抓包查看信息
yum install tcpdump -y
tcpdump -i eth0 host -nn 172.20.76.18
```

#### 4.2使用单播

- 把组播改成单播，不然都发向224，没用的信息造成宽带和资源的浪费。
- 组播修改成单播需要添加源地址和目标地址。

```bash
#在master上设置：172.20.76.18 
vim /etc/keepalived/keepalived.conf
#在global里面，关闭或者注释此选项
vrrp_strict #严格遵守VRRP协议,不允许状况:1,没有VIP地址,2.单播邻居,3.在VRRP版本2中有IPv6地址.

vrrp_instance VIP1 { 
    state MASTER            
    interface eth0
    virtual_router_id 76   
    priority 100 
    advert_int 2 
    unicast_src_ip 172.20.76.18    #源地址
    unicast_peer {				#目标地址
        172.20.76.28
    }   
    authentication {
      auth_type PASS
      auth_pass redhat
   }   
  virtual_ipaddress {
     172.20.76.187 dev eth0 label eth0:0                                              
  }   
}



#在slave上设置成单播：172.20.76.28 
#设置修改相同的地方：】
3注释
vrrp_strict #严格遵守VRRP协议,不允许状况:1,没有VIP地址,2.单播邻居,3.在VRRP版本2中有IPv6地址.

unicast_src_ip 172.20.76.28    #源地址
unicast_peer {				#目标地址
  172.20.76.18
    }   
#重启访问就单播的了
```



#### 4.3多组VIP（主主路由器）

- 工作中会设置多组VIP,分别在不通的keepalived上运行
- 单播下做实验

```bash
#在master：172.20.76.18  slave:172.20.76.18  设置主配置文件 
vim /etc/keepalived/keepalived.conf
# master：172.20.76.18
vrrp_instance VIP1 {
    state MASTER        
    interface eth0
    virtual_router_id 76   
    priority 100
   advert_int 2 
    unicast_src_ip 172.20.76.18
    unicast_peer {
        172.20.76.28
    }
    authentication {
      auth_type PASS
      auth_pass redhat
   }   
  virtual_ipaddress {
     172.20.76.187 dev eth0 label eth0:0    # 将VIP1的VIP写在这
     172.20.76.188 dev eth0 label eth0:1                                          
  }   
}

# BACKUP :172.20.76.18
vrrp_instance VIP2 {
    state BACKUP    
    interface eth0
    virtual_router_id 176 
    priority 80
    advert_int 2 
    unicast_src_ip 172.20.76.18
    unicast_peer {
        172.20.76.28
    }   
    authentication {
      auth_type PASS
      auth_pass redhat
   }   
  virtual_ipaddress {
     172.20.76.189 dev eth0 label eth0:2    #将VIP2的VIP全部写在这儿
     172.20.76.190 dev eth0 label eth0:3    
  }   
} 

#重启服务
systemctl restart keepalived.service

#在master：172.20.76.28  slave:172.20.76.28  设置主配置文件
vim /etc/keepalived/keepalived.conf
# BACKUP :172.20.76.18
vrrp_instance VIP1 {
    state BACKUP        
    interface eth0
    virtual_router_id 76   
    priority 80
    advert_int 2 
   unicast_src_ip 172.20.76.28    #源地址
   unicast_peer {               #目标地址
        172.20.76.18
  }   
    authentication {
      auth_type PASS
      auth_pass redhat
   }   
  virtual_ipaddress {
     172.20.76.187 dev eth0 label eth0:0                                          
     172.20.76.188 dev eth0 label eth0:1   
  }   
}
# MASTER:172.20.76.28
vrrp_instance VIP2 { 
   state MASTER            
   interface eth0
   virtual_router_id 176   
   priority 100 
   advert_int 2 
   unicast_src_ip 172.20.76.28    #源地址
   unicast_peer {               #目标地址
        172.20.76.18
  }   
   authentication {
      auth_type PASS
      auth_pass redhat
   }   
  virtual_ipaddress {
      172.20.76.189 dev eth0 label eth0:2                                              
      172.20.76.190 dev eth0 label eth0:3                                              
   }   
}       

#重启服务
systemctl restart keepalived.service


#查看
ifconfig
#测试 停掉一台机器,在查看ifconfig
systemctl stop keepalived

```



#### 4.4非抢占式

- 默认是工作在抢占式

- HA的实际运行过程中，当主机发生异常，且后期恢复正常后，存在抢占或非抢占两种情况。

  结合实际需求，可能有很多用户需要非抢占的HA工作模式。keepalived能够很好的支持这一需求。

- Keepalived不抢占有两种写法，第一种就是,下面的列子： 
       主 backup priority100    nopreempt 
        备 backup priority50
  
  -  nopreempt #关闭VIP抢占，需要VIP state都为BACKUP	
- 第二种就是讲优先级设置成一样的，这样后面启动的不会抢占。 
        主 backup priority100 
       备 backup priority100

```bash
#设置配置文件： master:172.20.76.18
vim /etc/keepalived/keepalived.conf 
vrrp_instance VIP1 {
    state BACKUP    #将master改成backup
    interface eth0
    virtual_router_id 76   
    priority 100 
   advert_int 2 
   nopreempt    #添加这一行，#关闭VIP抢占，需要VIP state都为BACKUP
    unicast_src_ip 172.20.76.18
    unicast_peer {
        172.20.76.28
    }
    authentication {
      auth_type PASS
      auth_pass redhat
   }   
  virtual_ipaddress {
     172.20.76.187 dev eth0 label eth0:0                                          
  }   
}

#重启服务
systemctl restart keepalived.service


#设置配置文件： slave:172.20.76.28
vim /etc/keepalived/keepalived.conf 
vrrp_instance VIP1 {
    state BACKUP    
    interface eth0
    virtual_router_id 76   
    priority 50
    advert_int 2 
   unicast_src_ip 172.20.76.28    #源地址
   unicast_peer {               #目标地址
        172.20.76.18
  }   
    authentication {
      auth_type PASS
      auth_pass redhat
   }   
  virtual_ipaddress {
     172.20.76.187 dev eth0 label eth0:0                               
     }   
}

#重启服务
systemctl restart keepalived.service
```

#### 4.5Keepalived邮箱通知配置

- 解决linux不能发送有邮件到邮箱的问题

```bash
#安装发送邮件的安装包
yum install mailx.x86_64 sendmail  -y
#发邮件给某个邮箱,不能发送成功
echo "mail test" |mail -s "linux xiangzi" lqlixl.com
#添加通知配置
cat /etc/mail.rc
	set from=805226454@qq.com
	set smtp=smtp.qq.com
	set smtp-auth-user=805226454@qq.com
	set smtp-auth-password=rlgsiminwaxtbbid   #QQ邮箱里面有设置
	set smtp-auth=login
	set ssl-verify=ignore
#在此发送邮件，邮箱就能收到
echo "mail test" |mail -s "linux xiangzi" lqlixl.com
```

- 可以发送邮件，设置keepalived发送邮件。
- Keepalived通知脚本

```bash
cat /etc/keepalived/notify.sh 
 #!/bin/bash
 contact='805226454@qq.com'
 notify() {
 mailsubject="$(hostname) to be $1, vip 转移"
 mailbody="$(date +'%F %T'): vrrp transition, $(hostname) changed to be $1"
 echo "$mailbody" | mail -s "$mailsubject" $contact
 }
 case $1 in
 master)
 notify master
 ;;
 backup)
 notify backup
 ;;
 fault)
 notify fault
 ;;
 *)
 echo "Usage: $(basename $0) {master|backup|fault}"
 exit 1
 ;;
 esac
 #给脚本加执行权限
 chmod a+x /etc/keepalived/notify.sh 
```

- Keepalived通知配置

```bash
#脚本的调用方法：
notify_master "/etc/keepalived/notify.sh master"
notify_backup "/etc/keepalived/notify.sh backup"
notify_fault "/etc/keepalived/notify.sh fault"

#重启服务
systemctl restart keepalived.service 
#发送邮件
bash /etc/keepalived/notify.sh master
```

![1560132068419](C:\Users\L\AppData\Roaming\Typora\typora-user-images\1560132068419.png)

- Keepalived通知验证
  - 停止keepalived服务，验证IP 切换后是否收到通知邮件：

#### 4.6设置三套keepalived+haproxy的服务

- 将172.20.76.28的复制到另一台机器上，修改配置就可以实现。
- 第一台挂机，去第二台，第二台挂机去第三台。

###  5.keepalived的实际应用

#### 5.1keepalived注意

- keepalived,的端口 是80，后端服务器的端口必须也是80，而且是同一局域网段，通过mac地址通信的，跨网段就会被交换机改变mac地址。

#### 5.2virtual server的定义

  ```bash
  virtual_server IP port #定义虚拟主机IP地址及其端口
  virtual_server fwmark int #IPVS的防火墙打标，实现基于防火墙的负载均衡集群
  virtual_server group string #将多个虚拟服务器定义成组，将组定义成虚拟服务
  ```

  ```bash
  #用法
  virtual_server IP port
  {
  	...
  	real_server {
  	...
  	}
  	…
  }
  
  delay_loop <INT> #检查后端服务器的时间间隔
  lb_algo rr|wrr|lc|wlc|lblc|sh|dh #定义调度算法
  lb_kind NAT|DR|TUN #集群的类型
  persistence_timeout <INT> #持续连接时长
  protocol TCP|UDP|SCTP #指定服务协议
  sorry_server <IPADDR> <PORT> #所有RS故障时，备用服务器地址
  real_server <IPADDR> <PORT>
  {
      weight <INT>  RS权重
      notify_up <STRING>|<QUOTED-STRINNG> #RS上线通知脚本
      notify_down <STRING>|<QUOTEN-STRING> #RS下线通知脚本
      HTTP_GET|SSL_GET|TCP_CHECK|SMTP_CHECK|MISC_CHECK{...} #定义当前主机的健康状态检测方法
     }
  ```

#### 5.3 应用层监测

  - HTTP_GET|SSL_GET：应用层检测

    ```bash
    HTTP_GET|SSL_GET {
    	url {
    		path <URL_PATH> #定义要监控的URL
    		status_code <INT> #判断上述检测机制为健康状态的响应码
    	}
    connect_timeout <INTEGER> #连接请求的超时时长
    nb_get_retry <INT> #重试次数
    delay_before_retry <INT> #重试之前的延迟时长
    connect_ip <IP ADDRESS> #向当前RS哪个IP地址发起健康状态检测请求
    connect_port <PORT> #向当前RS的哪个PORT发起健康状态检测请求
    bindto <IP ADDRESS> #发出健康状态检测请求时使用的源地址
    bind_port <PORT> #发出健康状态检测请求时使用的源端口
    }
    ```

#### 5.4TCP监测

- 传输层检测 TCP_CHECK

```bash
TCP_CHECK {
connect_ip <IP ADDRESS>：#向当前RS的哪个IP地址发起健康状态检测请求
connect_port <PORT>：#向当前RS的哪个PORT发起健康状态检测请求
bindto <IP ADDRESS>：#发出健康状态检测请求时使用的源地址
bind_port <PORT>：#发出健康状态检测请求时使用的源端口
connect_timeout <INTEGER>：#连接请求的超时时长
}
```

#### 5.5keepalived与LVS DR

- 架构规划

- ![1560138444564](C:\Users\L\AppData\Roaming\Typora\typora-user-images\1560138444564.png)

- 划分网段服务

  ```bash
  #构建服务
  LB  keepalive：172.20.76.18   172.20.76.28 LB  
  后端服务器：172.20.76.38   172.20.76.48     
  错误服务器：172.20.76.58   
  ```

  - 设置keepalive的配置

```bash

#在 keepalive：172.20.76.18   172.20.76.28配置文件在主配置文件里定义子配置文件位置，创建一个子配置文件。
vim  /etc/keepalived/keepalived.conf
  ...
  notify_master "/etc/keepalived/notify.sh master"
  notify_backup "/etc/keepalived/notify.sh backup"
  notify_fault "/etc/keepalived/notify.sh fault"
}         
 include /etc/keepalived/conf/*.conf   
 
mkdir /etc/keepalived/conf
vim /etc/keepalived/conf/web.conf

#在  172.20.76.18 keepalive master上设置
virtual_server 172.20.76.18 80 {
delay_loop 6
lb_algo wrr 
lb_kind DR
#persistence_timeout 120 #会话保持时间
protocol TCP 
                                                                                 
sorry_server 172.20.76.58 80

real_server 172.20.76.38 80 {
    weight 1
    TCP_CHECK {
    connect_timeout 5
    nb_get_retry 3
    delay_before_retry 3
    connect_port 80
     }   
 }

real_server 172.20.76.48 80 {
 weight 1
 TCP_CHECK {
 connect_timeout 5
 nb_get_retry 3
 delay_before_retry 3
 connect_port 80
   }   
  }
}

#在  172.20.76.28 keepalive backup 上设置
virtual_server 172.20.76.28 80 {
delay_loop 6
lb_algo wrr 
lb_kind DR
#persistence_timeout 120 #会话保持时间
protocol TCP 
                                                                                 
sorry_server 172.20.76.58 80

real_server 172.20.76.38 80 {
    weight 1
    TCP_CHECK {
    connect_timeout 5
    nb_get_retry 3
    delay_before_retry 3
    connect_port 80
     }   
 }

real_server 172.20.76.48 80 {
 weight 1
 TCP_CHECK {
 connect_timeout 5
 nb_get_retry 3
 delay_before_retry 3
 connect_port 80
   }   
  }
}

#重启服务
 systemctl restart keepalived.service 
 #查看规则
 yum install ipvsadm.x86_64 -y
 ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  172.20.76.28:80 wrr
  -> 172.20.76.38:80              Route   1      0          0         
  -> 172.20.76.48:80              Route   1      0          0 
 
 
```

- 后端web服务器配置

  ```bash
  #在172.20.76.38，172.20.76.48，172.20.76.58，上都跑一次
  #安装前先安装
  yum install net-tools -y
  #借助脚本生成VIP，内容如下
  vim lvs_dr_rs.sh
  #!/bin/bash
  vip=172.20.76.187
  mask='255.255.255.255'
  dev=lo:1
  
  case $1 in
  start)
      echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore
      echo 1 > /proc/sys/net/ipv4/conf/lo/arp_ignore
      echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce
      echo 2 > /proc/sys/net/ipv4/conf/lo/arp_announce
      ifconfig $dev $vip netmask $mask #broadcast $vip up
      #route add -host $vip dev $dev
      echo "The RS Server is Ready!"
      ;;
  stop)
      ifconfig $dev down
   	echo 0 > /proc/sys/net/ipv4/conf/all/arp_ignore
      echo 0 > /proc/sys/net/ipv4/conf/lo/arp_ignore
      echo 0 > /proc/sys/net/ipv4/conf/all/arp_announce
      echo 0 > /proc/sys/net/ipv4/conf/lo/arp_announce
      echo "The RS Server is Canceled!"
      ;;
  *)
      echo "Usage: $(basename $0) start|stop"
      exit 1
      ;;
  esac
  #运行脚本生成VIP
  bash lvs_dr_rs.sh start
  172.20.76.58:
  echo 38 >/var/www/html/index.html
  172.20.76.48
  echo 48 >/var/www/html/index.html
  #在172.20.76.58上写sorry界面，前面38和48都挂了
  echo sorry 58 >/var/www/html/index.html
  ```

  - 访问测试

    ```bash
    curl http://172.20.76.187/
    38
    curl http://172.20.76.187/
    48
    curl http://172.20.76.187/
    38
    curl http://172.20.76.187/
    48
    #关闭master节点的keepalived服务继续测试
    curl http://172.20.76.187/
    38
    curl http://172.20.76.187/
    48
    ```

- 修改为http监测（主备都要配置）
  
  - 为了防止服务、端口都启动但是服务无效的情况，直接监测web服务的某个界面是否可用

```bash
virtual_server 172.20.76.187 80 {
        delay_loop 6
        lb_algo wrr
        lb_kind DR
        #persistence_timeout 120 #会话保持时间
        protocol TCP
		
	sorry_server 172.20.76.58 80
		
        real_server 172.20.76.38 80 {
        weight 1
        HTTP_GET {
        url {
                path /test/index.html
                status_code 200
        }
        connect_timeout 5
        nb_get_retry 3
        delay_before_retry 3
        #connect_port 80
        }
   }
    real_server 172.20.76.48 80 {
        weight 1
        HTTP_GET {
        url {
                path /test/index.html #监测web服务器的此界面
                status_code 200 #返回状态码为200才确认该服务存活，不然就踢出组
        }
        connect_timeout 5
        nb_get_retry 3
        delay_before_retry 3
        #connect_port 80
        }
   }
}
```

#### 5.6VRRP script

- keepalived调用外部的辅助脚本进行资源监控，并根据监控的结果状态能实现优先动态调整
- vrrp_script:自定义资源监控脚本，vrrp实例根据脚本返回值进行下一步操作，脚本可被多个实例调用。
  track_script:调用vrrp_script定义的脚本去监控资源，定义在实例之内，调用事先定义的vrrp_script
- 分两步骤

```bash
 #(1).先定义一个脚本
 vrrp_script <SCRIPT_NAME> {
	script <STRING>|<QUOTED-STRING>
	OPTIONS
	}
#(2) 调用此脚本
track_script {
	SCRIPT_NAME_1
	SCRIPT_NAME_2
	}
```

```bash
#keepalived 配置文件里面的格式	
vrrp_script <SCRIPT_NAME> { #定义一个检测脚本，在global_defs 之外配置
	script <STRING>|<QUOTED-STRING> # shell命令或脚本路径
	interval <INTEGER> # 间隔时间，单位为秒，默认1秒
	timeout <INTEGER> # 超时时间
	weight <INTEGER:-254..254> # 权重，监测失败后会执行权重+操作
	fall <INTEGER> #脚本几次失败转换为失败
	rise <INTEGER> # 脚本连续监测成果后，把服务器从失败标记为成功的次数
	user USERNAME [GROUPNAME] # 执行监测的用户或组
	init_fail # 设置默认标记为失败状态，监测成功之后再转换为成功状态
}
```

##### 5.6.1文件存在来判断检测keepalived服务是否可用

```bash
#在keepalived中定义检测脚本（在vrrp_instance之上定义，主备配置相同）
[root@centos ~]$vim /etc/keepalived/keepalived.conf
vrrp_script chk_down { #定义脚本名称
        script "/bin/bash -c '[[ -f /etc/keepalived/down ]]' && exit 7 || exit 0" #检测文件/etc/keepalived/down是否存在，若存在返回7，说明服务不可用（此处必须再借助一个脚本当检测服务不可用时生成此文件），不存在返回0，说明服务可用
        interval 2
        weight -50 #若检测不成功则优先级-50
        fall 3
        rise 5
        timeout 2
}
#在vrrp_instance中最后面调用脚本（主备配置相同）
track_script {
        chk_down
        }
#访问测试
手动创建/etc/keepalived/down文件，查看master的VIP是否漂移至backup
#touch /etc/keepalived/down
```

##### 5.6.2检测网络通不通来判断服务可不可用

```bash
#创建检测脚本 172.20.76.18
[root@centos ~]$vim /etc/keepalived/ping.sh
#!/bin/bash
ping -c 2 172.20.76.18 &> /dev/null #假设网关永久可用，ping网关不通则证明本身服务不可用
#ping -c 2 172.20.76.180 &> /dev/null #模拟本身服务不可用
if [ $? -eq 0 ];then
        exit 0 #若能ping通则返回状态码0
else
        exit 2 #若ping不同则返回状态码2
fi
[root@centos ~]$chmod +x /etc/keepalived/ping.sh
#在keepalived中定义检测脚本（在vrrp_instance之上定义，主备配置相同）
[root@centos ~]$vim /etc/keepalived/keepalived.conf
vrrp_script linux_ping { #定义脚本名称
        script /etc/keepalived/ping.sh #检测脚本位置
        interval 2
        weight -50 #若检测不成功则优先级-50
        fall 3
        rise 5
        timeout 2
}
#在vrrp_instance中最后面调用脚本（主备配置相同）
track_script {
        linux_ping
        }
#访问测试
在脚本中改变网关为192.168.300.1模拟keepalived服务宕机，查看VIP是否漂移至backup
```

#### 5.7keepalived+haproxy

- 5.6 前面使用的LVS+keepalive搭建的检测
- 使用keepalived和haproxy搭建高可用性检测

```bash
#创建脚本
[root@centos ~]$vim /etc/keepalived/chk_haproxy.sh
#!/bin/bash
killall -0 haproxy #通过检测haproxy进程是否存活，通过信息来判断。
echo $?
[root@centos ~]$chmod +x /etc/keepalived/chk_haproxy.sh
#keepalived中定义检测脚本
vrrp_script chk_haproxy {
    script /etc/keepalived/chk_haproxy.sh
    interval 1
    weight -50
    fall 3 
    rise 5
    timeout 2
}
#调用检测脚本
track_script {
     chk_haproxy
     }
 [root@centos ~]$systemctl restart keepalived
 #访问测试
 手动关闭master的haproxy服务，查看VIP是否迁移
```

#### 5.8高可用性nginx

```bash
检测，高可用nginx与高可用haproxy同理，将检测脚本改成killall -0 nginx即可
```

































