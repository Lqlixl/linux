

-network bridge=mybr0   生成一个桥接网卡 br0，虚拟机的桥接网卡都指向这儿。相当于交换机一样。一头   出口指向物理机的eth0，另一头指向n个虚拟机.就像一个公司或者教室,所有电脑都是指向交换机,与交换机相连接,在通过交换机到指向路由器上面或者某个网络上面.网关旧在路由器这儿

宿主机没有桥接地址。地址配置在br0上。



# KVM

## 1.  简介

- KVM 是Kernel-based Virtual Machine的简称，是一个开源的系统虚拟化模块，自Linux 2.6.20之后集成在Linux的各个主要发行版本中。
- KVM使用Linux自身的调度器进行管理，所以相对于Xen（https://zhuanlan.zhihu.com/p/33324585），其核心源码很少。
- KVM目前已成为学术界的主流VMM之一。 
- KVM的虚拟化需要硬件支持（如Intel VT技术或者AMD V技术)。是基于硬件的完全虚拟化。而Xen早期则是基于软件模拟的Para-Virtualization，新版本则是基于硬件支持的完全虚拟化。但Xen本身有自己的进程调度器，存储管理模块等，所以代码较为庞大。
- 广为流传的商业系统虚拟化软件VMware ESXI系列是Full-Virtualization，IBM文档：http://www.ibm.com/developerworks/cn/linux/l-using-kvm/

```bash
#Guest
  客户机系统，包括CPU（vCPU）、内存、驱动（Console、网卡、I/O 设备驱动等），被KVM置于一种受限制的CPU模式下运行。

#KVM
运行在内核空间，提供 CPU 和内存的虚级化，以及客户机的 I/O拦截，Guest的部分I/O被KVM拦截后，交给 QEMU处理.

#QEMU
  修改过的被KVM虚机使用的QEMU代码，运行在用户空间，提供硬件I/O虚拟化，通过IOCTL/dev/kvm设备和KVM交互，但是，KVM本身不执行任何硬件模拟，需要用户空间程序通过 /dev/kvm 接口设置一个客户机虚拟服务器的地址空间，向它提供模拟I/O，并将它的视频显示映射回宿主的显示屏。目前这个应用程序是QEMU.
```

![1560653282507](D:\学习资料\markdow\KVM.assets\1560653282507.png)

# 2.  宿主机环境准备

- KVM需要宿主机CPU必须支持虚拟化功能，因此如果是在vmware workstation上使用虚拟机做宿主机，那么必须要在虚拟机配置界面的处理器选项中开启虚拟机化功能。架构

### 2.0 实验架构

![1560734240727](D:\学习资料\markdow\KVM.assets\1560734240727.png)



### 2.1 CPU开启虚拟化

- CPU需要开启虚拟化,宿主机内存不能太小,设置四个网卡,两个桥接,两个仅主机.

![1560653629731](D:\学习资料\markdow\KVM.assets\1560653629731.png)

#### 2.1.1 查看CPU

```bash
#查看宿主机CPU是否在硬件上支持虚拟化扩展特性
[root@CentOS7 ~]#grep -E "vmx|svm" /proc/cpuinfo | wc -l
4
```

### 2.2  安装KVM工具包

- kvm 是模块是基于内核空间的,运行模块已经子啊内核里了,安装的是管理命令.

```bash
#最小化系统安装基础命令
yum install  vim iotop bc gcc gcc-c++ glibc glibc-devel pcre pcre-devel openssl  openssl-devel zip unzip zlib-devel  net-tools lrzsz tree ntpdate telnet lsof tcpdump wget libevent libevent-devel bc  systemd-devel bash-completion traceroute -y


# 1.安装
yum install qemu-kvm qemu-kvm-tools libvirt virt-manager virt-install -y

#2. 开启libvirtd,并设置为开机启动.
systemctl start libvirtd
systemctl enable libvirtb

#3. libvirt 程序包是一个与虚拟机监控程序相独立的虚拟化应用程序接口，它可以与操作系统的一系列虚拟化性能进行交互。

# 查看; nat网卡,只能访问外网,不能回来.
[root@CentOS7 ~]#ifconfig
virbr0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 192.168.122.1  netmask 255.255.255.0  broadcast 192.168.122.255
        ether 52:54:00:94:1a:5e  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

# 4. 修改virbr0的地址
[root@CentOS7 ~]#grep 192.168.122.1 /etc/ -R
/etc/libvirt/qemu/networks/autostart/default.xml:  <ipaddress='192.168.122.1' netmask='255.255.255.0'>
/etc/libvirt/qemu/networks/default.xml:  <ip address='192.168.122.1' netmask='255.255.255.0'>
#进入/etc/libvirt/qemu/networks/autostart/default.xml 就可以修改了,下面那个是软连接,(小巷子没修改)



```

### 2.3 创建NAT网络虚拟机

#### 2.3.1 创建磁盘

```bash
 qemu-img -h

#创建磁盘 
#lqlixl1台机器
create 创建文件;-f 格式;raw 存放位置,size 初始化大小 一般为10G; 
[root@CentOS7 ~]#qemu-img create -f raw /var/lib/libvirt/images/centos.raw 10G 
Formatting '/var/lib/libvirt/images/centos.raw', fmt=raw size=10737418240 

[root@CentOS7 ~]#ll -h /var/lib/libvirt/images/centos.raw 
-rw-r--r-- 1 root root 10G Jun 16 19:21 /var/lib/libvirt/images/centos.raw

#lqlixl2台机器


```

#### 2.3.2 官网下载最小化镜像

```bash
#安装命令
yum install wget -y

#进入官网找最小化镜像下载
cd /usr/local/src
wget http://mirrors.aliyun.com/centos/7.6.1810/isos/x86_64/CentOS-7-x86_64-Minimal-1810.iso

#查看
[root@CentOS7 src]#ll /usr/local/src/
total 940032
-rw-r--r-- 1 root root 962592768 Nov 26  2018 CentOS-7-x86_64-Minimal-1810.iso
```



#### 2.3.3  virsh-install命令使用帮助

```bash
#查看安装帮助
[root@CentOS7 src]#virt-install --help

usage: virt-install --name NAME --ram RAM STORAGE INSTALL [options]
使用指定安装介质新建虚拟机。
optional arguments:
-h, --help      #show this help message and exit
--version       #show program's version number and exit
--connect URI   #使用 libvirt URI 连接到 hypervisor

#安装提示
[root@CentOS7 src]#virt-install 
ERROR    
--name is required  #必须有名字
--memory amount in MiB is required  内存大小
--disk storage must be specified (override with --disk none)  指定磁盘存储
An install method must be specified  安装指定方法
(--location URL, --cdrom CD/ISO, --pxe, --import, --boot hd|cdrom|...)


#通用安装选项
 --name NAME   #客服端事件名称.
 --vcpus VCPUS # 为虚拟机配置的 vcpu 数
 --memory MEMORY #配置虚拟机内存分配.
 --cdrom CDROM  #光驱的安装介质
 --disk path DISK  #使用不同选项指定村粗.
 --network NETWORK #配置虚拟网络接口
 --graphics vnc  # 配置虚拟机显示设置
 --virt-type kvm,qemu,xen #使用的管理程序名称
 --autostart #引导主机时自动启动域,宿主因为断电等原因关机,在此开机后,虚拟机是否自动开机.
 ----noautoconsole #不要自动尝试连接到客户端控制台

--virt-type kvm \
--name linux \
--vcpus 2 \
--memory 1024 \
--cdrom=/usr/local/src/CentOS-7-x86_64-Minimal-1810.iso \
--disk path=/var/lib/libvirt/images/centos.raw \
--network network=default \
--graphics vnc,listen=0.0.0.0 \
--noautoconsole \
--autostart

#创建网络虚拟机
[root@CentOS7 src]#virt-install --virt-type kvm \
> --name linux \
> --vcpus 2 \
> --memory 1024 \
> --cdrom=/usr/local/src/CentOS-7-x86_64-Minimal-1810.iso \
> --disk path=/var/lib/libvirt/images/centos.raw \
> --network network=default \
> --graphics vnc,listen=0.0.0.0 \
> --noautoconsole \
> --autostart

Starting install...
Domain installation still in progress. You can reconnect to 
the console to complete the installation process.
```

#### 2.3.4 配置网络

- 配置宿主机 192.168.1.18 的网络

```bash
#配置网卡前,CentOS7.2版本之前需要安装一个包,将网卡eth0和eth1绑定bind1需要安装.
yum install bridge-utils -y

#进入目录
[root@CentOS7 src]#cd /etc/sysconfig/network-scripts/

# 注意:配置前,要去查看四个网卡位置,两个仅主机配置在bond1上面,两个桥接配置在bond0上.

#配置 bond0
[root@CentOS7 network-scripts]#vim ifcfg-bond0
BOOTPROTO=static
NAME=bond0
DEVICE=bond0
ONBOOT=yes
BONDING_MASTER=yes
BONDING_OPTS="mode=1 miimon=100"
BRIDGE=br0

mode 1高可用性
mode 2
mode 3
mode 4

#配置 br0   桥接的
[root@CentOS7 network-scripts]#vim ifcfg-br0
TYPE=Bridge
BOOTPROTO=static
NAME=br0
DEVICE=br0
ONBOOT=yes
IPADDR=172.20.76.200
NETMASK=255.255.0.0
GATEWAY=172.20.0.1 


#配置 eth1    绑定在bond0
[root@CentOS7 network-scripts]#vim ifcfg-eth1
DEVICE=eth1
NAME=eth1
ONBOOT=yes
#NM_CONTROLLED=no
MASTER=bond0
#USERCTL=no
SLAVE=yes  

#配置 eth2 绑定在bond0
[root@CentOS7 network-scripts]#cp ifcfg-eth1 ifcfg-eth2
DEVICE=eth2
NAME=eth2
ONBOOT=yes
#NM_CONTROLLED=no
MASTER=bond0
#USERCTL=no
SLAVE=yes   



#配置br1   仅主机
[root@CentOS7 network-scripts]#vim ifcfg-br1
TYPE=Bridge
BOOTPROTO=static
NAME=br1
DEVICE=br1
ONBOOT=yes
IPADDR=192.168.1.100
NETMASK=255.255.0.0  

#配置 bond1   
[root@CentOS7 network-scripts]#vim ifcfg-bond1
BOOTPROTO=static
NAME=bond1
DEVICE=bond1
ONBOOT=yes
BONDING_MASTER=yes
BONDING_OPTS="mode=1 miimon=100"
BRIDGE=br1   



#配置 eth0 绑定在bond1 
[root@CentOS7 network-scripts]#vim ifcfg-eth1
DEVICE=eth0
NAME=eth0
ONBOOT=yes
#NM_CONTROLLED=no
#USERCTL=no
MASTER=bond1
SLAVE=yes  



#配置 eth3  绑定在bond1
[root@CentOS7 network-scripts]#cp ifcfg-eth1 ifcfg-eth3
DEVICE=eth3
NAME=eth3
ONBOOT=yes
#NM_CONTROLLED=no
MASTER=bond1
#USERCTL=no
SLAVE=yes   



#重启网卡
[root@CentOS7 network-scripts]#systemctl restart network

#查看网卡
yum install net-tools -y
[root@CentOS7 ~]#ifconfig
bond0: flags=5187<UP,BROADCAST,RUNNING,MASTER,MULTICAST>  mtu 1500
        inet6 fe80::250:56ff:fe35:f00b  prefixlen 64  scopeid 0x20<link>
        ether 00:50:56:35:f0:0b  txqueuelen 1000  (Ethernet)
        RX packets 379689  bytes 286449266 (273.1 MiB)
        RX errors 0  dropped 69  overruns 0  frame 0
        TX packets 27  bytes 2306 (2.2 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

bond1: flags=5187<UP,BROADCAST,RUNNING,MASTER,MULTICAST>  mtu 1500
        inet6 fe80::250:56ff:fe33:d209  prefixlen 64  scopeid 0x20<link>
        ether 00:50:56:33:d2:09  txqueuelen 1000  (Ethernet)
        RX packets 715  bytes 120716 (117.8 KiB)
        RX errors 0  dropped 2  overruns 0  frame 0
        TX packets 80  bytes 10826 (10.5 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

br0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.20.76.200  netmask 255.255.0.0  broadcast 172.20.255.255
        inet6 fe80::250:56ff:fe33:d209  prefixlen 64  scopeid 0x20<link>
        ether 00:50:56:35:f0:0b  txqueuelen 1000  (Ethernet)
        RX packets 32465  bytes 8358726 (7.9 MiB)
        RX errors 0  dropped 67  overruns 0  frame 0
        TX packets 23  bytes 1802 (1.7 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

br1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.100  netmask 255.255.0.0  broadcast 192.168.255.255
        inet6 fe80::250:56ff:fe3b:2ae9  prefixlen 64  scopeid 0x20<link>
        ether 00:50:56:33:d2:09  txqueuelen 1000  (Ethernet)
        RX packets 457  bytes 69244 (67.6 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 76  bytes 10058 (9.8 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth0: flags=6211<UP,BROADCAST,RUNNING,SLAVE,MULTICAST>  mtu 1500
        ether 00:50:56:33:d2:09  txqueuelen 1000  (Ethernet)
        RX packets 621  bytes 87302 (85.2 KiB)
        RX errors 0  dropped 2  overruns 0  frame 0
        TX packets 97  bytes 12082 (11.7 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth1: flags=6211<UP,BROADCAST,RUNNING,SLAVE,MULTICAST>  mtu 1500
        ether 00:50:56:35:f0:0b  txqueuelen 1000  (Ethernet)
        RX packets 361988  bytes 279981421 (267.0 MiB)
        RX errors 0  dropped 69  overruns 0  frame 0
        TX packets 27  bytes 2306 (2.2 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth2: flags=6211<UP,BROADCAST,RUNNING,SLAVE,MULTICAST>  mtu 1500
        ether 00:50:56:35:f0:0b  txqueuelen 1000  (Ethernet)
        RX packets 20641  bytes 7361302 (7.0 MiB)
        RX errors 0  dropped 6  overruns 0  frame 0
        TX packets 15  bytes 1166 (1.1 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth3: flags=6211<UP,BROADCAST,RUNNING,SLAVE,MULTICAST>  mtu 1500
        ether 00:50:56:33:d2:09  txqueuelen 1000  (Ethernet)
        RX packets 113  bytes 35234 (34.4 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 3  bytes 227 (227.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 3  bytes 227 (227.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

virbr0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.122.1  netmask 255.255.255.0  broadcast 192.168.122.255
        ether 52:54:00:94:1a:5e  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

vnet0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet6 fe80::fc54:ff:fe87:3134  prefixlen 64  scopeid 0x20<link>
        ether fe:54:00:87:31:34  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 205 overruns 0  carrier 0  collisions 0



#配置完宿主机IP改变.重新连接xshell

```



### 2.4通过virt-manager管理虚拟机

```bash
[root@CentOS7 network-scripts]#virt-manager 
[root@CentOS7 network-scripts]#libGL error: unable to load driver: swrast_dri.so
libGL error: failed to load driver: swrast

```



![1560676406308](D:\学习资料\markdow\KVM.assets\1560676406308.png)



安装的时候按tab键,写  net.names=0 boisdevname=0  作用是进入安装虚拟机后网卡名是eth开头的.

#### 2.4.1 添加网卡

```bash
#添加外网网卡
vim /etc/sysconfig/network-scripts/ifcfg-eth1
DEVICE=eth1                                                 
NAME=eth1
BOOTPROTO=static
IPADDR=172.20.76.201
PREFIX=16
GATEWAY=172.20.0.1
ONBOOT=yes
DNS1=223.5.5.5
DNS2=223.6.6.6


#内网网卡


#仅主机 eth0
[root@localhost ~]# cat /etc/sysconfig/network-scripts/ifcfg-eth0
DEVICE=eth0                              
NAME=eth0
BOOTPROTO=static
IPADDR=192.168.1.101
PREFIX=16
ONBOOT=yes


```





### 2.5  创建另一台内网虚拟机

- 直接复制刚刚创建的虚拟机的磁盘.

```bash
#进入目录存放磁盘的目录
[root@CentOS7 images]#ll /var/lib/libvirt/images/centos.raw 
-rw-r--r-- 1 root root 10737418240 Jun 17 01:47 centos.raw

#拷贝一份10G磁盘
[root@CentOS7 images]#cp centos.raw centos1.raw 
[root@CentOS7 images]#ls

#创建网络虚拟机
[root@CentOS7 images]#virt-install --virt-type kvm \
> --name linux1 \
> --vcpus 2 \
> --memory 1024 \
> --cdrom=/usr/local/src/CentOS-7-x86_64-Minimal-1810.iso \
> --disk path=/var/lib/libvirt/images/centos1.raw \
> --network network=default \
> --graphics vnc,listen=0.0.0.0 \
> --noautoconsole \
> --autostart

Starting install...
Domain installation still in progress. You can reconnect to 
the console to complete the installation process.

#通过登录查看,有两台机器了
[root@CentOS7 images]#virt-manager 


#双击进入虚拟机管理界面以后直接关机虚拟机,然后在重启,force off.

#添加网卡

#仅主机模式
[root@localhost network-scripts]# cat /etc/sysconfig/network-scripts/ifcfg-eth0 
NAME=eth0
DEVICE=eth0
ONBOOT=yes
IPADDR=192.168.1.102
BOOTPROTO=static
NETMASK=255.255.0.0

#重启服务
systemctl restart network

```



## 3  第二台宿主机搭建

打开一台新的虚拟机

```bash
#默认保存路径
[root@CentOS7 src]#ll /var/lib/libvirt/images/

#创建一个格式为qcow2大小为10G的裸磁盘(qcow2 不会一开始有10G，随着写入增加，存放数据混乱的)
qemu-img create -f qcow2 /var/lib/libvirt/images/centos.qcow2 10G

#在官网下载最小化安装的系统
wget http://mirrors.aliyun.com/centos/7.6.1810/isos/x86_64/CentOS-7-x86_64-Minimal-1810.iso

#创建网络虚拟机
[root@CentOS7 src]#virt-install --virt-type kvm \
--name python \
--vcpus 2 \
--memory 1024 \
--cdrom=/usr/local/src/CentOS-7-x86_64-Minimal-1810.iso \
--disk path=/var/lib/libvirt/images/centos.qcow2  \
--network network=default \
--graphics vnc,listen=0.0.0.0 \
--noautoconsole \
--autostart 	

Starting install...
Domain installation still in progress. You can reconnect to 
the console to complete the installation process.

```

### 3.1 配置网络

#### 3.1.1 创建外网

```bash
#网络
#配置bond0
[root@CentOS7 network-scripts]#cat  /etc/sysconfig/network-scripts/ifcfg-bond0
BOOTPROTO=static
NAME=bond0
DEVICE=bond0
ONBOOT=yes
BONDING_MASTER=yes
BONDING_OPTS="mode=1 miimon=100"
BRIDGE=br0

#配置bond1
[root@CentOS7 network-scripts]#cat  /etc/sysconfig/network-scripts/ifcfg-bond1
BOOTPROTO=static
NAME=bond1
DEVICE=bond1
ONBOOT=yes
BONDING_MASTER=yes
BONDING_OPTS="mode=1 miimon=100"
BRIDGE=br1

#配置br0
[root@CentOS7 network-scripts]#cat  /etc/sysconfig/network-scripts/ifcfg-br0
TYPE=Bridge
BOOTPROTO=static
NAME=br0
DEVICE=br0
ONBOOT=yes
IPADDR=172.20.76.202
NETMASK=255.255.0.0
GATEWAY=172.20.0.1
DNS1=172.20.0.1
DNS2=223.5.5.5

#配置br1
[root@CentOS7 network-scripts]#cat  /etc/sysconfig/network-scripts/ifcfg-br1
TYPE=Bridge
BOOTPROTO=static
NAME=br1
DEVICE=br1
ONBOOT=yes
IPADDR=192.168.1.105
NETMASK=255.255.0.0

#配置eth0
[root@CentOS7 network-scripts]#cat  /etc/sysconfig/network-scripts/ifcfg-eth0
DEVICE=eth0
NAME=eth0
ONBOOT=yes
MASTER=bond1
BOOTPROTO=static
SLAVE=yes

#配置eth1
[root@CentOS7 network-scripts]#cat  /etc/sysconfig/network-scripts/ifcfg-eth1
DEVICE=eth1
NAME=eth1
ONBOOT=yes
MASTER=bond0
BOOTPROTO=static
SLAVE=yes

#配置eth2
[root@CentOS7 network-scripts]#cat  /etc/sysconfig/network-scripts/ifcfg-eht2
DEVICE=eth2
NAME=eth2
ONBOOT=yes
MASTER=bond1
BOOTPROTO=static
SLAVE=yes

#配置eth3
[root@CentOS7 network-scripts]#cat  /etc/sysconfig/network-scripts/ifcfg-eth3
DEVICE=eth3
NAME=eth3
ONBOOT=yes
MASTER=bond0
BOOTPROTO=static
SLAVE=yes

#重启网络
systemctl restart network
```

### 3.2 管理安装虚拟机

```bash
#虚拟机管理命令：
[root@s1 src]# virsh list #列出当前开机的
[root@s1 src]# virsh list --all 3列出所有
[root@s1 src]# virsh shutdown CentOS-7-x86_64 #正常关机
[root@s1 src]# virsh start CentOS-7-x86_64 #正常关机
[root@s1 src]# virsh destroy centos7 #强制停止/关机
[root@s1 src]# virsh undefine Win_2008_r2-x86_64 #强制删除
[root@s1 src]# virsh autostart centos7 #设置开机自启动


#管理虚拟机
virt-manager

进入后直接 net.ifnames=0 biosdevname=0 网卡名

装系统。

创建完成
```



### 3.3 配置虚拟机里的网络

```bash
#配置能连接外网的的虚拟机
需要添加一个网卡。

#仅主机 eth0
[root@localhost ~]# cat /etc/sysconfig/network-scripts/ifcfg-eth0
DEVICE=eth0                              
NAME=eth0
BOOTPROTO=static
IPADDR=192.168.1.106
PREFIX=16
ONBOOT=yes

#桥接 eth1
[root@localhost ~]# cat /etc/sysconfig/network-scripts/ifcfg-eth1
DEVICE=eth1
NAME=eth1
BOOTPROTO=static
IPADDR=172.20.76.203
PREFIX=16
GATEWAY=172.20.0.1
ONBOOT=yes
DNS1=223.5.5.5
DNS2=223.6.6.6

#重启网络
systemctl restart network
OK

#需要安装一个apaceh 服务。等会直接复制这台机器作为另一台内网服务器使用
```

### 3.4 第二台宿主机的第二台内网虚拟机

#### 3.4.1 安装虚拟机

```bash
#复制磁盘
[root@CentOS7 images]#ll /var/lib/libvirt/images/
total 2791620
-rw-r--r-- 1 qemu qemu 1429340160 Jun 17 22:10 centos.qcow2
[root@CentOS7 src]#cd /var/lib/libvirt/images/
[root@CentOS7 images]#cp centos.qcow2 centos1.qcow2
[root@CentOS7 images]#ll
total 2791620
-rw-r--r-- 1 qemu qemu 1429340160 Jun 17 22:03 centos1.qcow2
-rw-r--r-- 1 qemu qemu 1429340160 Jun 17 22:10 centos.qcow2

#安装新的虚拟机
root@CentOS7 network-scripts]#virt-install --virt-type kvm \
> --name python1 \
> --vcpus 2 \
> --memory 1024 \
> --cdrom=/usr/local/src/CentOS-7-x86_64-Minimal-1810.iso \
> --disk path=/var/lib/libvirt/images/centos1.qcow2  \
> --network network=default \
> --graphics vnc,listen=0.0.0.0 \
> --noautoconsole \
> --autostart

#管理和开机
virt-manager

双击进入界面，然后关机，在开机

```

#### 3.4.2 配置第二台虚拟机网络

```bash
#仅主机模式
[root@localhost network-scripts]# cat /etc/sysconfig/network-scripts/ifcfg-eth0 
NAME=eth0
DEVICE=eth0
ONBOOT=yes
IPADDR=192.168.1.107
BOOTPROTO=static
NETMASK=255.255.0.0

#重启服务
systemctl restart network

测试 ping 192.168.1.105

```





































































































































































































- 配置另一个机器192.168.1.28

![1560674807434](D:\学习资料\markdow\KVM.assets\1560674807434.png)



```bash
#因为四个网卡顺序问题需要重新调eth0,eth1,eth2,eth3.或者在设置网卡的时候,顺序调制好.

#在192.168.1.18上
scp ifcfg-eth* ifcfg-b* 192.168.1.28:/etc/sysconfig/network-scripts/
root@192.168.1.28's password: 
ifcfg-bond0                                                                    100%  115    63.7KB/s   00:00    
ifcfg-bond1                                                                    100%  115    56.4KB/s   00:00    
ifcfg-br0                                                                      100%  136    98.4KB/s   00:00    
ifcfg-br1                                                                      100%  102    61.7KB/s   00:00 
ifcfg-eth0                                                                     100%   73    79.8KB/s   00:00    
ifcfg-eth1                                                                     100%   73    56.7KB/s   00:00       
ifcfg-eth2                                                                     100%   73    78.1KB/s   00:00    
ifcfg-eth3                                                                     100%   73    63.3KB/s   00:00

#在192.168.1.28 上

[root@CentOS7 ~]#cd /etc/sysconfig/network-scripts/

#修改拷贝过来的文件修改后

#br0
[root@CentOS7 network-scripts]#cat ifcfg-br0
TYPE=Bridge
BOOTPROTO=static
NAME=br0
DEVICE=br0
ONBOOT=yes
IPADDR=172.20.76.101
NETMASK=255.255.0.0
GATEWAY=172.20.0.1
DNS1=172.20.0.1

#br1
[root@CentOS7 network-scripts]#cat ifcfg-br1
TYPE=Bridge
BOOTPROTO=static
NAME=br1
DEVICE=br1
ONBOOT=yes
IPADDR=192.168.1.101
NETMASK=255.255.0.0
 
 #配置eth0
[root@CentOS7 network-scripts]#cat ifcfg-eth0
DEVICE=eth0
NAME=eth0
ONBOOT=yes
MASTER=bond1
BOOTPROTO=static
SLAVE=yes

#配置eth1
[root@CentOS7 network-scripts]#cat ifcfg-eth1
DEVICE=eth1
NAME=eth1
ONBOOT=yes
MASTER=bond0
BOOTPROTO=static
SLAVE=yes

#配置eth2
[root@CentOS7 network-scripts]#cat ifcfg-eth2
DEVICE=eth2
NAME=eth2
ONBOOT=yes
MASTER=bond1
BOOTPROTO=static
SLAVE=yes

#配置eth3
[root@CentOS7 network-scripts]#cat ifcfg-eth3
DEVICE=eth3
NAME=eth3
ONBOOT=yes
MASTER=bond0
BOOTPROTO=static
SLAVE=yes

#重启网卡
[root@CentOS7 network-scripts]#systemctl restart network

#查看
[root@CentOS7 ~]#ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master bond1 state UP group default qlen 1000
    link/ether 00:0c:29:65:d4:8c brd ff:ff:ff:ff:ff:ff
3: eth1: <BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master bond0 state UP group default qlen 1000
    link/ether 00:0c:29:65:d4:82 brd ff:ff:ff:ff:ff:ff
4: eth2: <BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master bond1 state UP group default qlen 1000
    link/ether 00:0c:29:65:d4:8c brd ff:ff:ff:ff:ff:ff
5: eth3: <BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master bond0 state UP group default qlen 1000
    link/ether 00:0c:29:65:d4:82 brd ff:ff:ff:ff:ff:ff
6: br1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 00:0c:29:65:d4:8c brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.101/16 brd 192.168.255.255 scope global noprefixroute br1
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe65:d48c/64 scope link 
       valid_lft forever preferred_lft forever
7: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 00:0c:29:65:d4:82 brd ff:ff:ff:ff:ff:ff
    inet 172.20.76.101/16 brd 172.20.255.255 scope global noprefixroute br0
       valid_lft forever preferred_lft forever
    inet6 fe80::64fa:deff:fe8e:e7be/64 scope link 
       valid_lft forever preferred_lft forever
8: bond0: <BROADCAST,MULTICAST,MASTER,UP,LOWER_UP> mtu 1500 qdisc noqueue master br0 state UP group default qlen 1000
    link/ether 00:0c:29:65:d4:82 brd ff:ff:ff:ff:ff:ff
9: bond1: <BROADCAST,MULTICAST,MASTER,UP,LOWER_UP> mtu 1500 qdisc noqueue master br1 state UP group default qlen 1000
    link/ether 00:0c:29:65:d4:8c brd ff:ff:ff:ff:ff:ff



```























- 架构

  ```bash
  #关闭防火墙
  systemctl disable firewalld
  
  #后端web服务
  #192.168.1.102
  
  yum install httpd -y
  
  echo 192.168.1.102>/var/www/html/index.html
  
  systemctl start httpd
  
  ss -ntl
  
  # 192.168.1.102
  
  yum install httpd -y
  
  echo 192.168.1.102>/var/www/html/index.html
  
  systemctl start httpd
  
  ss -ntl
  
  #keepalive+haproxy
  
  yum install keepaive haproxy -y
vim /etc/keepalived/keepalived.conf
  master：keepalived 的配置
  vim /etc/keepalived/keepalived.conf
  ...
  #   vrrp_strict
   vrrp_iptables
  ...
  
  vrrp_instance VIP1 {
      state MASTER
      interface eth0
      virtual_router_id 76
      priority 100 
      advert_int 3
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
  
  #backup :keepalived 的配置
  vim /etc/keepalived/keepalived.conf
  ...
  #   vrrp_strict
   vrrp_iptables
   ...
  vrrp_instance VIP1 {
      state BACKUP 
      interface eth0
      virtual_router_id 76
      priority 50
      advert_int 3
      unicast_src_ip 172.20.76.28
      unicast_peer {
        172.20.76.18
      }   
  
      authentication {
          auth_type PASS
          auth_pass redhat
      }   
      virtual_ipaddress {
          172.20.76.187 dev eth0 label eth0:0
      }   
      
      
      #配置文件与后方服务端连接,两台机器一样的。
  vim /etc/haproxy/haproxy.conf
  listen web_port_80
      bind 172.20.76.187:80   
      mode http
      server 172.20.76.38 172.20.76.38:80 check inter 3s fall 3 rise 5
      server 172.20.76.38 172.20.76.48:80 check inter 3s fall 3 rise 5
  
  cat /etc/sysctl.conf 
  net.ipv4.ip_nonlocal_bind = 1
  net.ipv4.ip_forward = 1 
  sysctl -p
  
  systemctl restart keepalived 
  systemctl restart haproxy
  
  ```
  
  



































