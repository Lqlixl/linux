## 一、高可用集群概念

- 集群类型
  - LB lvs/nginx 负载均衡
  - HA 高可用，解决单点故障问题
  - HPC 高性能集群
- 系统可用性：SLA

指标（95%）=(60x24x30)x(1-0.9995)

- 系统故障：
  - 硬件故障：设计缺陷、wear out （损耗）、自然灾害等
  - 软件故障：设计缺陷

- 提升系统高可用性的解决方案之降低MTTR（平均故障时间）

解决方案：建立冗余机制

active/passive 主/备
active/active 双主
active --> HEARTBEAT --> passive
active <--> HEARTBEAT <--> active

- 高可用的是"服务"

- shared storage：

  - NAS(Network Attached Storage)：网络附加存储，基于网络的共享文件系统
  - SAN(Storage Area Network)：存储区域网络，基于网络的块级别的共享

- 网络分区、隔离设备

- 双节点集群

  辅助设备：ping node, quorum disk（仲裁设备）

  - failover：故障切换，即某资源的主节点故障时，将资源转至其他节点的操作
  - failback：故障移回，即某资源的主节点故障后重新修改上线后，将之前已转移至其他节点的资源重新切回的过程

- HA Cluster实现方案

  AIS应用程序接口规范：

  ​	RHCS：Red Hat Cluster Suite红帽集群套件
  ​	heartbeat：基于心跳监测实现服务高可用
  ​	pacemaker+corosync：资源管理与故障转移
  ​	vrrp(Virtual Router Redundancy Protocol)：虚拟路由冗余协议,解决静态网关单点风险
  ​	软件层—keepalived
  ​	物理层—路由器、三层交换机

## 二、Keepalived

### - 2.1：keepalived简介

- vrrp协议的软件实现，原生设计目的是为了高可用ipvs服务
- 功能：
  - 基于vrrp协议完成地址流动
  - 为vip地址所在的节点生成ipvs规则（在配置文件中预先定义）
  - 为ipvs集群的各RS做健康状态检测
  - 基于脚本调用接口通过执行脚本完成脚本中定义的功能，进而影响集群事务，以此支持nginx、haproxy等服务

- 组件：

  - 用户空间核心组件：
    - vrrp stack	VIP消息通告
    - checkers      监测real server
    - system call    标记real server权重
    - SMTP  邮件组件
    - ipvs wrapper   生成IPVS规则
    - Netlink Reflector   网络接口
    - WatchDog  监控进程
  - 控制组件：配置文件分析器
  - IO复用器
  - 内存管组件

- 术语：

  - 虚拟路由器：Virtual Router
  - 虚拟路由器标识：VRID（0-255），唯一标识虚拟路由器
  - 物理路由器：
    - master：主设备
    - backup：备用设备
    - priority：优先级
  - VIP：Virtual IP
  - VMAC：Viryual MAC（***）

- 通告：心跳，优先级等；周期性

- 工作方式：抢占式，非抢占式

- 安全工作：

  - 认证：无认证、简单字符认证：预共享密钥

- 工作模式：

  主/备：单虚拟路由器

  主/主：主/备（虚拟路由器1），备/主虚拟路由器（2）

### - 2.2：keepalived安装

```bash
#keepalived安装
yum install keepalived  (CentOS)
apt-get install keepalived  (Ubuntu)
#程序环境
/etc/keepalived/keepalived.conf  #主配置文件
/usr/sbin/keepalived    #主程序文件
```

### - 2.3：keepalived配置

```bash
Global definitions #全局配置
VRRP instance #一个虚拟路由器
Virtual server group 
Virtual server #ipvs集群和rs
```

#### - 2.3.1：vrrp_instance配置

```bash
state MASTER/BACKUP #当前及诶单在此虚拟路由器上的初始状态
interface IFACE_NAME #绑定为当前虚拟路由器使用的物理接口
virtual_router_id VRID #当前虚拟路由器唯一标识，范围是0-255
priority 100 #当前物理节点在此虚拟路由器中的哦呦现价，范围是1-254
advert_int 1 #vrrp通告时间间隔，默认1s
authentication {#认证机制
    auth_type AH|PASS
    auth_pass <PASSWORD> 仅前8位有效
}
virtual_ipaddress {#虚拟IP
	<IPADDR>/<MASK> brd <IPADDR> dev <STRING> scope <SCOPE> label <LABEL>
	192.168.30.6/24 dev eth1:1
}
track_interface{#配置监控网络接口，一旦出现故障，则转为FAULT状态实现地址迁移
    eth0
}
```

#### - 2.3.2：组播配置示例-MASTER

```bash
global_defs {
    notification_email {
        root@localhost #keepalived发生故障切换时邮件发送的对象，可以按行区分写多个
    }
    notification_email_from keepalived@localhost
    smtp_server 127.0.0.1
    smtp_connect_timeout 30
    router_id test1
    vrrp_skip_check_adv_addr #所有报文都检查比较消耗性能，此配置为收到的报文和上一个报文是同一个路由器则跳过检查报文中的源地址
    vrrp_strict #严格遵守VRRP协议,不允许状况:1,没有VIP地址,2.单播邻居,3.在VRRP版本2中有IPv6地址
   vrrp_garp_interval 0 #ARP报文发送延迟 
   vrrp_gna_interval 0 #消息发送延迟
   #vrrp_mcast_group4 224.0.0.18 #默认组播IP地址，224.0.0.0到239.255.255.255
   #vrrp_iptables #禁止自动生成防火墙策略
}

vrrp_instance VIP1 {
    state MASTER
    interface ens33
    virtual_router_id 30
	priority 100
	advert_int 2 #默认为1s，可根据生产实际改动
	
	authentication {
        auth_type PASS
	    auth_pass 1111
	}
	virtual_ipaddress {
        192.168.30.248 dev ens33 label ens33:0
	}
}
```

#### - 2.3.4：组播配置示例-BACKUP

```bash
global_defs {
    notification_email {
        root@localhost
    }
    notification_email_from keepalived@localhost
	smtp_server 127.0.0.1
	smtp_connect_timeout 30
	router_id test2
	vrrp_skip_check_adv_addr #
	vrrp_strict #严格遵守VRRP协议。
	vrrp_garp_interval 0 #ARP报文发送延迟
	vrrp_gna_interval 0 #消息发送延迟
	#vrrp_mcast_group4 224.0.0.18 #组播IP地址，224.0.0.0到239.255.255.255
	#vrrp_iptables #禁止自动生成防火墙策略
}

vrrp_instance VIP1 {
    state BACKUP
	interface ens33
	virtual_router_id 30
	priority 80
	advert_int 2
	
	authentication {
        auth_type PASS
        auth_pass 1111
	}
	virtual_ipaddress {
        192.168.30.248 dev ens33 label ens33:0
	}
}
```

- 测试：启动服务，查看VIP是否出现，手动操作master宕机，查看VIP是否迁移到backup
- 注意：route_id必须不同，virtual_router_id必须相同

#### - 2.3.5：单播配置示例

- 组播占用网络带宽，影响网络性能，单播既能实现监控又能节省网络资源
- 多个keepalied之间互相单播，可以有更高的可用性

```bash
unicast_src_ip 本机源
unicast_peer {
   目标主机IP
}
#配置示例
vrrp_instance VIP1 {#单播配置必须关闭global中的vrrp_strict选项，不然服务起不来
    state MASTER
    interface ens33
    virtual_router_id 30
    priority 100
    advert_int 2
    unicast_src_ip 192.168.30.16
    unicast_peer {
       192.168.30.6
    } #backup配置相同
```

#### - 2.3.6：非抢占

- nopreempt #关闭VIP抢占，需要VIPstate都设置为BACKUP，配置在优先级高的主机上

```bash
vrrp_instance VIP1 {
    state BACKUP
	interface ens33
	virtual_router_id 30
	priority 100
	advert_int 2
}

vrrp_instance VIP1 {
    state BACKUP
	interface ens33
	virtual_router_id 30
	priority 80
	advert_int 2
}
```

#### - 2.3.7：keepalived双主配置

两个或以上VIP分别运行在不同的keepalived服务器，以实现服务器并行提供
web访问的目的，提高服务器资源利用率

```bash
#A主机配置
vrrp_instance VIP1 {
    state BACKUP
    interface ens33
    virtual_router_id 30
    priority 80
    advert_int 2
    unicast_src_ip 192.168.30.6
    unicast_peer {
        192.168.30.16
    }
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.30.248/24 dev ens33 label ens33:0
    }
}

vrrp_instance VIP2 {
    state MASTER
    interface ens33
    virtual_router_id 37
    priority 100
    advert_int 2
    unicast_src_ip 192.168.30.6
    unicast_peer {
        192.168.30.16
    }
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.30.249/24 dev ens33 label ens33:1
    }
}
#B主机配置
vrrp_instance VIP1 {
    state MASTER
    interface ens33
    virtual_router_id 30
    priority 100
    advert_int 2
    unicast_src_ip 192.168.30.16
    unicast_peer {
        192.168.30.6
    }
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.30.248/24 dev ens33 label ens33:0
    }
}

vrrp_instance VIP2 {
    state BACKUP
    interface ens33
    virtual_router_id 37
    priority 80
    advert_int 2
    unicast_src_ip 192.168.30.16
    unicast_peer {
        192.168.30.6
    }
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.30.249/24 dev ens33 label ens33:1
    }
}
```

#### - 2.3.8：keepalived通知配置

- 发件人配置：

```bash
[root@centos ~]$yum install mailx -y
[root@centos ~]$vim /etc/mail.rc
set from=1146992442@qq.com
set smtp=smtp.qq.com
set smtp-auth-user=1146992442@qq.com
set smtp-auth-password=poydgckfrtcbgagj #此码在邮箱设置账户中开启pop3功能
set smtp-auth=login
set ssl-verify=ignore
#测试
[root@centos ~]$echo "mail test" | mail -s "linux" 1146992442@qq.com
```

- nopreempt：定义工作模式为非抢占模式
- preempt_delay 300：抢占模式，节点上线后触发新选举操作的延时时长，默认模式
- 定义通知脚本：

```bash
notify_master <string>|<QUOTED-STRING>：
#当前节点成为主节点时触发的脚本
notify_backup <STRING>|<QUOTED-STRING>：
#当前节点转为备节点时出发的脚本
notify_fault <STRING>|<QUOTED-STRING>：
#当前节点转为"失败"状态时触发的脚本
notify <STRING>|<QUOTED-STRING>：
#通用格式的通知触发机制，一个脚本可完成以上三种状态转换时的通知
```

- 通知脚本内容：

```bash
[root@centos ~]$vim /etc/keepalived/chk.sh
#!/bin/bash
contact='1146992442@qq.com'
notify() {
mailsubject="$(hostname) to be $1, vip  转移"
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
```

- 脚本调用方法：

```bash
#加在最后面即可
...
virtual_ipaddress {
        192.168.30.249/24 dev ens33 label ens33:1
    }
}
notify_master "/etc/keepalived/chk.sh master"
notify_backup "/etc/keepalived/chk.sh backup"
notify_fault "/etc/keepalived/chk.sh fault"
```

- 验证：停止keepalived服务，验证IP切换后是否收到通知邮件

### - 2.4：keepalived与IPVS

- 虚拟服务器配置参数
- virtual server的定义：

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

#### - 2.4.1：应用层监测

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

#### - 2.4.2：TCP监测

- 传输层检测TCP_CHECK

```bash
TCP_CHECK {
	connect_ip <IP ADDRESS> #向当前RS的哪个IP地址发起健康状态检测请求
	connect_port <PORT> #向当前RS的哪个PORT发起健康状态检测请求
	bindto <IP ADDRESS> #发出健康状态检测请求时使用的源地址
	bind_port <PORT> #发出健康状态检测请求时使用的源端口
	connect_timeout <INTEGER> #连接请求的超时时长
}
```

#### - 2.4.3：示例1-实现keepalived+LVS

- 架构规划

![1559960646165](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1559960646165.png)

- 思路：

  keepalived主备之间的横向监测和keepalived对后端web服务器的纵向监测

- keepalived配置（主备都要配置）

```bash
[root@centos ~]$vim /etc/keepalived/keepalived.conf
! Configuration File for keepalived

global_defs {
   notification_email {
   root@localhost
}
   notification_email_from root@linux.com
   smtp_server 127.0.0.1
   smtp_connect_timeout 30
   router_id 16
   vrrp_skip_check_adv_addr
   #vrrp_strict
   #vrrp_iptables
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}

vrrp_instance VIP1 {
    state MASTER
    interface ens33
    virtual_router_id 30
    priority 100
    advert_int 2
    unicast_src_ip 192.168.30.16
    unicast_peer {
        192.168.30.6
    }
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.30.248/24 dev ens33 label ens33:0
    }
notify_master "/etc/keepalived/chk.sh master"
notify_backup "/etc/keepalived/chk.sh backup"
notify_fault "/etc/keepalived/chk.sh fault"
}
include /etc/keepalived/conf/*.conf

[root@centos ~]$vim /etc/keepalived/conf/web1.conf
virtual_server 192.168.30.248 80 {
        delay_loop 5 #检测间隔时间
        lb_algo wrr #调度算法
        lb_kind DR  #指定集群类型为DR模式
        #persistence_timeout 120 #会话保持时间
        protocol TCP #指定服务协议

        real_server 192.168.30.26 80 {
        weight 1
        TCP_CHECK {
        connect_timeout 5 #连接请求的超时时长
        nb_get_retry 3
        delay_before_retry 3
        connect_port 80
        }
   }

        real_server 192.168.30.36 80 {
        weight 1
        TCP_CHECK {
 		connect_timeout 5
        nb_get_retry 3
        delay_before_retry 3
        connect_port 80
        }
   }
}
[root@centos ~]$ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.30.248:80 wrr
  -> 192.168.30.26:80             Route   1      0          0         
  -> 192.168.30.36:80             Route   1      0          0 
```

- web服务器配置（web服务器都要配置）

```bash
#借助脚本生成VIP，内容如下
[root@centos ~]$vim lvs_dr_rs.sh
#!/bin/bash
vip=192.168.30.248
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
[root@centos ~]$bash lvs_dr_rs.sh start
```

- 访问测试

```bash
[root@centos ~]$curl 192.168.30.248
this is apache
[root@centos ~]$curl 192.168.30.248
this is nginx
[root@centos ~]$curl 192.168.30.248
this is apache
[root@centos ~]$curl 192.168.30.248
this is nginx
#关闭master节点的keepalived服务继续测试
[root@centos ~]$curl 192.168.30.248
this is apache
[root@centos ~]$curl 192.168.30.248
this is nginx
```

- 修改为http监测（主备都要配置）

为了防止服务、端口都启动但是服务无效的情况，直接监测web服务的某个界面是否可用

```bash
virtual_server 192.168.30.248 80 {
        delay_loop 6
        lb_algo wrr
        lb_kind DR
        #persistence_timeout 120 #会话保持时间
        protocol TCP

        real_server 192.168.30.26 80 {
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
	    real_server 192.168.30.36 80 {
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

#### - 2.4.4：VRRP script

##### - 2.4.4.1：配置介绍

- keepalived调用外部的辅助脚本进行资源监控，并根据根据监控的状态结果能实现优先动态调整

- vrrp_script：

  自定义资源监控脚本，vrrp实例根据脚本返回值进行下一步操作，脚本可被多个实例调用

- track_script：调用vrrp_script定义的脚本去监控资源，定义在实例之内，调用事先定义的vrrp_script

- 分为两个步骤：

```bash
#（1）定义脚本 
vrrp_script <SCRIPT_NAME> {
	script <STRING>|<QUOTED-STRING>
	OPTIONS
}
#（2）调用脚本
track_script {
	SCRIPT_NAME_1
	SCRIPT_NAME_2
}
```

```bash
vrrp_script <SCRIPT_NAME> { #定义一个检测脚本，在global_defs 之外配置
	script <STRING>|<QUOTED-STRING> # shell命令或脚本路径
	interval <INTEGER> # 间隔时间，单位为秒，默认1秒
	timeout <INTEGER> # 超时时间
	weight <INTEGER:-254..254> # 权重，监测失败后会执行权重+操作
	fall <INTEGER> #脚本连续几次检测失败转换为失败
	rise <INTEGER> # 脚本连续监测成果后，把服务器从失败标记为成功的次数
	user USERNAME [GROUPNAME] # 执行监测的用户或组
	init_fail # 设置默认标记为失败状态，监测成功之后再转换为成功状态
}
```

```bash
vrrp_script chk_down { #基于第三方仲裁设备
	script "/bin/bash -c '[[ -f /etc/keepalived/down ]]' && exit 7 || exit 0"
	interval 1
	weight -80
	fall 3
	rise 5
	timeout 2
}

vrrp_instance VI_1 {
	…
	track_script {
	chk_down
	}
}
```

##### - 2.4.4.2：配置示例1-ping网关自证

```bash
#创建检测脚本
[root@centos ~]$vim /etc/keepalived/ping.sh
#!/bin/bash
ping -c 2 192.168.30.1 &> /dev/null #假设网关永久可用，ping网关不通则证明本身服务不可用
#ping -c 2 192.168.300.1 &> /dev/null #模拟本身服务不可用
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

##### - 2.4.4.3：配置示例2-文件存在与否检测keepalived服务是否可用

```bash
##在keepalived中定义检测脚本（在vrrp_instance之上定义，主备配置相同）
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
```

### 2.5：keepalived+haproxy

#### 2.5.1：高可用HAProxy

- 与2.4中的实验不同的是此处使用haproxy+keepalived来实现

```bash
#创建脚本
[root@centos ~]$vim /etc/keepalived/chk_haproxy.sh
#!/bin/bash
killall -0 haproxy #通过向haproxy服务发送消息来确认是否存活
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

#### 2.5.2：高可用nginx

高可用nginx与高可用haproxy同理，将检测脚本改成killall -0 nginx即可







