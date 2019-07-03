

# 部署 nova 控制节点与计算节点

## 1. 简介

- nova 是 openstack 最早的组件之一，nova 分为控制节点和计算节点，计算节点通过 novacomputer 进行虚拟机创建，通过 libvirt 调用 kvm 创建虚拟机，nova 之间通信通过 rabbitMQ队列进行通信，其组件和功能如下：
  - API：负责接收和响应外部请求。
  - Scheduler：负责调度虚拟机所在的物理机。
  - Conductor：计算节点访问数据库的中间件。
  - Consoleauth：用于控制台的授权认证。
  - Novncproxy：VNC 代理，用于显示虚拟机操作终端。



- 官方部署文档 ： https://docs.openstack.org/mitaka/zh_CN/install-guiderdo/common/get_started_compute.html



- Nova-API 的功能：
  Nova-api 组件实现了 restful API 的功能，接收和响应来自最终用户的计算 API 请求，接收外部的请求并通过 message queue 将请求发动给其他服务组件，同时也兼容 EC2 API，所以也可以使用 EC2 的管理工具对 nova 进行日常管理。
  - nova scheduler：
    - nova scheduler 模块在 openstack 中的作用是决策虚拟机创建在哪个主机（计算节点）上。决策一个虚拟机应该调度到某物理节点，需要分为两个步骤：
      过滤（filter），过滤出可以创建虚拟机的主机



##  2. 安装并配置 nova 控制节点：

- 官方文档： https://docs.openstack.org/ocata/zh_CN/install-guide-rdo/nova-controllerinstall.html

### 2.1 安装nova 控制端

```bash
#安装软件包：
yum install openstack-nova-api openstack-nova-conductor openstack-nova-console openstack-nova-novncproxy openstack-nova-scheduler openstack-nova-placement-api

# yum install openstack-nova-api openstack-nova-conductor openstacknova-console openstack-nova-novncproxy openstack-nova-scheduler openstack-novaplacement-api

```

### 2.2 准备数据库：

```bash
mysql -uroot -predhat

#创建数据库nova_api
CREATE DATABASE nova_api;
  

#授权nova对数据库nova_api权限
 GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'redhat';
 
#创建数据库nova
CREATE DATABASE nova;

#授权nova对数据库nova 的权限
 GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'redhat';
 
 
 #创建数据库nova_cell0
 CREATE DATABASE nova_cell0;
 
 #授权nova 对数据库nova_cell0的权限
 GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'redhat';
 
 
 #刷新权限
  flush privileges;
```



### 2.3 验证数据库

- 控制端

```bash
#登录mysql客服端
mysql -unova -h172.20.76.10 -predhat

show databases;
```



### 2.4 创建nova服务并注册

```bash
#环境配置
source /data/script/admin-ocata.sh 

# 查看你有没有参加创建用户和给 nova 用户添加 admin 角色：
openstack user create --domain default --password-prompt nova

openstack role add --project service --user nova admin

#创建nova服务，创建 nova 服务实体：
[root@control ~]# openstack service create --name nova --description "OpenStack Compute" compute
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Compute                |
| enabled     | True                             |
| id          | 2664a8bb8d944d1bb37708d6f47b314e |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+




```

#### 2.4.1 创建公共端点：创建 Compute 服务 API 端点 

```bash
[root@control ~]# openstack endpoint create --region RegionOne compute public http://openstack-lqlixl.net:8774/v2.1
+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | 68d4db6f36ec40489783a13f7b06ccc1         |
| interface    | public                                   |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | 2664a8bb8d944d1bb37708d6f47b314e         |
| service_name | nova                                     |
| service_type | compute                                  |
| url          | http://openstack-lqlixl.net:8774/v2.1 |
+--------------+------------------------------------------+



```



#### 2.4.2  创建私有端点，创建 Compute 服务 API 端点 

```bash

[root@control ~]# openstack endpoint create --region RegionOne compute internal http://openstack-lqlixl.net:8774/v2.1
+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | 7008c538c28f47b4902dffa4c96dea1a         |
| interface    | internal                                 |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | 2664a8bb8d944d1bb37708d6f47b314e         |
| service_name | nova                                     |
| service_type | compute                                  |
| url          | http://openstack-lqlixl.net:8774/v2.1 |
+--------------+------------------------------------------+


```



#### 2.4.3  创建管理端点：创建 Compute 服务 API 端点 

```bash

[root@control ~]# openstack endpoint create --region RegionOne compute admin http://openstack-lqlixl.net:8774/v2.1
+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | a7a4af369cd04833a7b41754868f33df         |
| interface    | admin                                    |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | 2664a8bb8d944d1bb37708d6f47b314e         |
| service_name | nova                                     |
| service_type | compute                                  |
| url          | http://openstack-lqlixl.net:8774/v2.1 |
+--------------+------------------------------------------+

```



### 2.5 创建 placement 用户并授权：

-  Placement 用户密码设置为 redhat

```bash
[root@control ~]# openstack user create --domain default --password-prompt placement
User Password:redhat
Repeat User Password:redhat
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | 8999a9638c0749f98d15dda10b402354 |
| enabled             | True                             |
| id                  | 6b8f272876854bf2bfe6c1b4704537d7 |
| name                | placement                        |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

```

#### 2.5.1 授权 admin 权限：

```bash
[root@control ~]# openstack role add --project service --user placement admin

#测试 查看
[root@control ~]# openstack user list
+----------------------------------+-----------+
| ID                               | Name      |
+----------------------------------+-----------+
| 429eb192cfae403392dec7708494d2a5 | demo      |
| 6b8f272876854bf2bfe6c1b4704537d7 | placement |
| 6dad69b6bd294885b5b596121d125818 | nova      |
| 82f134dff4e44dc888d16811025cd0ad | neutron   |
| 8a13ded5d07649a19f523973ae2dd6b2 | glance    |
| d239b2edce494c21a7b686037b23508d | admin     |
+----------------------------------+-----------+
[root@control ~]# openstack service list
+----------------------------------+----------+----------+
| ID                               | Name     | Type     |
+----------------------------------+----------+----------+
| 2664a8bb8d944d1bb37708d6f47b314e | nova     | compute  |
| 54904230fd73497494018a0964994480 | glance   | image    |
| f2c612c55a8844c7b9f59eebff84e9ad | keystone | identity |
+----------------------------------+----------+----------+

```

#### 2.5.2 创建placement API 并注册

```bash

[root@control ~]# openstack service create --name placement --description "Placement API" placement
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Placement API                    |
| enabled     | True                             |
| id          | 1aa90592f1b04d6da20457efcc73e1c4 |
| name        | placement                        |
| type        | placement                        |
+-------------+----------------------------------+


```



#### 2.5.3   创建公共端点

```bash
[root@control ~]# openstack endpoint create --region RegionOne placement public http://openstack-lqlixl.net:8778
+--------------+--------------------------------------+
| Field        | Value                                |
+--------------+--------------------------------------+
| enabled      | True                                 |
| id           | 5e32f2e100f84b95a500d5f2bf4f78ab     |
| interface    | public                               |
| region       | RegionOne                            |
| region_id    | RegionOne                            |
| service_id   | 1aa90592f1b04d6da20457efcc73e1c4     |
| service_name | placement                            |
| service_type | placement                            |
| url          | http://openstack-lqlixl.net:8778 |
+--------------+--------------------------------------+



```



#### 2.5.4 创建私有端点

```bash

[root@control ~]# openstack endpoint create --region RegionOne placement internal http://openstack-lqlixl.net:8778
+--------------+--------------------------------------+
| Field        | Value                                |
+--------------+--------------------------------------+
| enabled      | True                                 |
| id           | e75d0c7eb6c548279654387e640a8553     |
| interface    | internal                             |
| region       | RegionOne                            |
| region_id    | RegionOne                            |
| service_id   | 1aa90592f1b04d6da20457efcc73e1c4     |
| service_name | placement                            |
| service_type | placement                            |
| url          | http://openstack-lqlixl.net:8778 |
+--------------+--------------------------------------+

```



#### 2.5.5 创建管理端点

```bash

[root@control ~]# openstack endpoint create --region RegionOne placement admin http://openstack-lqlixl.net:8778
+--------------+--------------------------------------+
| Field        | Value                                |
+--------------+--------------------------------------+
| enabled      | True                                 |
| id           | 9ee900af7112423d8072672eba0501ec     |
| interface    | admin                                |
| region       | RegionOne                            |
| region_id    | RegionOne                            |
| service_id   | 1aa90592f1b04d6da20457efcc73e1c4     |
| service_name | placement                            |
| service_type | placement                            |
| url          | http://openstack-lqlixl.net:8778 |
+--------------+--------------------------------------+


```

### 2.6 编辑管理节点配置文件

#### 2.6.1 编辑``/etc/nova/nova.conf``文件

```bash

use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:123456@172.20.76.17
my_ip = 172.20.76.7
auth_strategy = keystone
connection = mysql+pymysql://nova:redhat@openstack-lqlixl.net/nova_api
connection = mysql+pymysql://nova:redhat@openstack-lqlixl.net/nova
api_servers = http://openstack-lqlixl.net:9292
auth_uri = http://openstack-lqlixl.net:5000
auth_url = http://openstack-lqlixl.net:35357
memcached_servers = openstack-lqlixl.net:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = redhat
virt_type = qemu
lock_path = /var/lib/nova/tmp
os_region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://openstack-lqlixl.net:35357/v3
username = placement
password = redhat
enabled = True
vncserver_listen = $my_ip
vncserver_proxyclient_address = $my_ip



# vim /etc/nova/nova.conf
#只启用计算和元数据API：
[DEFAULT]
...
enabled_apis = osapi_compute,metadata
#消息队列访问权限：#rabbit 服务器 IP，不支持代理
transport_url = rabbit://openstack:redhat@172.20.76.27
#配置my_ip来使用控制节点的管理接口的IP 地址。
my_ip = 172.20.76.7
#启用网络服务支持：
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

#配置数据库的连接
[api_database]
...
connection = mysql+pymysql://nova:redhat@openstack-lqlixl.net/nova_api

[database]
...
connection = mysql+pymysql://nova:redhat@openstack-lqlixl.net/nova

#配置认证服务访问：
[api]
...
auth_strategy = keystone

[keystone_authtoken]
...
auth_uri = http://openstack-lqlixl.net:5000
auth_url = http://openstack-lqlixl.net:35357
memcached_servers = openstack-lqlixl.net:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = redhat

#启用并配置远程控制台访问：
[vnc]
 ...
enabled = True
vncserver_listen = $my_ip
vncserver_proxyclient_address = $my_ip

#配置镜像服务 API 的位置
[glance]
...
api_servers = http://openstack-lqlixl.net:9292

#配置锁路径
[oslo_concurrency]
...
lock_path = /var/lib/nova/tmp


#配置Placement API:
[placement]
...
os_region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://openstack-lqlixl.net:35357/v3
username = placement
password = redhat


```

#### 2.6.2 官方出现几个版本的错误，修复

```bash
#必须通过将以下配置添加到以下内容来启用对Placement API的访问 /etc/httpd/conf.d/00-nova-placement-api.conf：

vim /etc/httpd/conf.d/00-nova-placement-api.conf
#最下方添加以下配置：
<Directory /usr/bin>
   <IfVersion >= 2.4>
      Require all granted
   </IfVersion>
   <IfVersion < 2.4>
      Order allow,deny
      Allow from all
   </IfVersion>
</Directory>

#重启httpd服务
```

#### 2.6.3 同步数据库

```bash
#同步Compute 数据库：
#nova_api 数据库
su -s /bin/sh -c "nova-manage api_db sync" nova

#nova cell0 数据库
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova

#nova cell1 数据库
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
577b75f4-2247-4d6a-8e5e-7b29825d7a67

#nova 数据库
su -s /bin/sh -c "nova-manage db sync" nova


#验证 nova cell0 和 nova cell1 是否正常注册：
[root@control ~]# nova-manage cell_v2 list_cells
+-------+--------------------------------------+
|  Name |                 UUID                 |
+-------+--------------------------------------+
| cell0 | 00000000-0000-0000-0000-000000000000 |
| cell1 | 577b75f4-2247-4d6a-8e5e-7b29825d7a67 |
+-------+--------------------------------------+

```



### 2.7 启服务，开机自启

- 启动 Compute 服务并将其设置为随系统启动：

```bash
#开机自启
systemctl enable openstack-nova-api.service openstack-nova-consoleauth.service openstack-nova-scheduler.service openstack-nova-conductor.service openstack-nova-novncproxy.service
	
#开机启动
systemctl start openstack-nova-api.service openstack-nova-consoleauth.service openstack-nova-scheduler.service openstack-nova-conductor.service openstack-nova-novncproxy.service


#写一个脚本将重启服务放在脚本里
# cat nova-restart.sh
#!/bin/bash
systemctl restart openstack-nova-api.service openstack-nova-consoleauth.service openstack-nova-scheduler.service openstack-nova-conductor.service openstack-nova-novncproxy.service

#加执行权限
chmod a+x nova-restart.sh
```

###  2.8 查看日志和rabbitmq连接

```bash
#日志
控制端有/var/log/http/*
lb:/var/log/haproxy.log

```

![1561290075259](D:\学习资料\markdow\openstacknova4.assets\1561290075259.png)



### 2.9 验证nova控制端

```bash
#验证 nova 控制端：


```

## 3. 部署 nova 计算节点

- 在计算节点服务器部署

- 如何在计算节点上安装并配置计算服务。计算服务支持多种虚拟化方式 [*hypervisors*](https://docs.openstack.org/mitaka/zh_CN/install-guide-rdo/common/glossary.html#term-hypervisor) to deploy [*instances*](https://docs.openstack.org/mitaka/zh_CN/install-guide-rdo/common/glossary.html#term-instance) or [*VMs*](https://docs.openstack.org/mitaka/zh_CN/install-guide-rdo/common/glossary.html#term-virtual-machine-vm). For simplicity, this configuration uses the [*QEMU*](https://docs.openstack.org/mitaka/zh_CN/install-guide-rdo/common/glossary.html#term-quick-emulator-qemu) hypervisor with the :term:[`](https://docs.openstack.org/mitaka/zh_CN/install-guide-rdo/nova-compute-install.html#id1)KVM <kernel-based VM (KVM)>`计算节点需支持对虚拟化的硬件加速。对于传统的硬件，本配置使用generic qumu的虚拟化方式。

#### 3.9.1 安装软件包

```bash
#安装 nova 计算服务
yum install openstack-nova-compute
```

#### 3.9.2  编辑``/etc/nova/nova.conf``文件

```bash
vim /etc/nova/nova.conf
# 配置RabbitMQ息队列的连接：
[DEFAULT]
...
rpc_backend = rabbit

[oslo_messaging_rabbit]
...
rabbit_host = openstack-lqlixl.net
rabbit_userid = openstack
rabbit_password = redhat
...
#配置认证服务访问：
[DEFAULT]
...
auth_strategy = keystone
[keystone_authtoken]
...
auth_uri = http://openstack-lqlixl.net:5000
auth_url = http://openstack-lqlixl.net:35357
memcached_servers = openstack-lqlixl.net:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = redhat
#配置 my_ip 选项
[DEFAULT]
...
my_ip = 172.20.76.27

















# 打开其他窗口查看是否支持，虚拟机硬件加速，返回0就需要添加下面的
egrep -c '(vmx|svm)' /proc/cpuinfo

#配置 libvirt 来使用 QEMU 去代替 KVM
[libvirt]
# ...
virt_type = qemu
```













































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































