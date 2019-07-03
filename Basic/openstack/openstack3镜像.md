# 部署镜像服务 glance：

- Glance 是 OpenStack 镜像服务组件，glance 服务默认监听在 9292 端口，其接收 REST API 请求，然后通过其他模块（glance-registry 及 image store）来完成诸如镜像的获取、上传、删除等操作，Glance 提供 restful API 可以查询虚拟机镜像的 metadata，并且可以获得镜像，通过Glance，虚拟机镜像可以被存储到多种存储上，比如简单的文件存储或者对象存储（比如OpenStack 中 swift 项目）是在创建虚拟机的时候，需要先把镜像上传到 glance，对镜像的列出镜像、删除镜像和上传镜像都是通过 glance 进行理，glance 有两个主要的服务，一个是glace-api 接收镜像的删除上传和读取，一个是 glance-Registry。
- glance-registry 负责与 mysql 数据交互，用于存储或获取镜像的元数据（metadata），提供镜像元数据相关的 REST 接口，通过 glance-registry 可以向数据库中写入或获取镜像的各种数据，glance-registyr 监听的端口是 9191，glance 数据库中有两张表，一张是 glance 表，一张是 imane property 表，image 表保存了镜像格式、大小等信息，image property 表保存了镜像的定制化信息。
  image store 是一个存储的接口层，通过这个接口 glance 可以获取镜像，image store 支持的存储有 Amazon 的 S3、openstack 本身的 swift、还有 ceph、glusterFS、sheepdog 等分布式存储，image store 是镜像保存与读取的接口，但是它只是一个接口，具体的实现需要外部的支持，glance 不需要配置消息队列，但是需要配置数据库和 keystone。
- 官方部署文档：https://docs.openstack.org/mitaka/zh_CN/install-guide-rdo/glance.html

- 创建的镜像  会同步一个， 然后数据信息，名字。大小等存放在数据库

  镜像存放在NFS

  

## 1. 先决条件

- 控制端

```bash
 #控制端安装glance
 yum install -y openstack-glance
 

```



```bash
#数据库MySQL
[root@rabbitsqlcache ~]# mysql -uroot -predhat


#创建 glance数据库
create database glance;

#对glance数据库授予恰当的权限
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'redhat';


#测试，控制端测试能不能连接数据库
mysql -uglance -h192.168.1.10 -predhat
show databases;
```

## 2.创建账户glance

- 控制端

```bash
#凭证来获取只有管理员能执行的命令的访问权限：
source /data/script/admin-ocata.sh

#创建glance用户：
openstack user create --domain default --password-prompt glance
User Password:redhat
Repeat User Password:redhat


#查看用户存在不存在
openstack user list

#添加glance用户到service 项目。并且赋予admin角色权限
openstack role add --project service --user glance admin

#创建glance服务实体，名字，描述，类型image
openstack service create --name glance --description "OpenStack Image" image

#查看实体
openstack service list

#查看注册的实体
openstack endpoint list

#9292端口是和用户交互相关的


```



###  2.1 注册实体

- 创建镜像服务的 API 端点

#### 2.1.1 创建 glance 服务：

```bash
[root@control script]#  openstack service create --name glance --description "OpenStack Image" image
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Image                  |
| enabled     | True                             |
| id          | a1245ec5c33041678b1f4bf5098739e2 |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+

```



#### 2.1.2 创建公有 endpoint：

```bash
[root@control script]# openstack endpoint create --region RegionOne image public http://openstack-lqlixl.net:9292
+--------------+--------------------------------------+
| Field        | Value                                |
+--------------+--------------------------------------+
| enabled      | True                                 |
| id           | c0733bd1c868431585566e385a9109d0     |
| interface    | public                               |
| region       | RegionOne                            |
| region_id    | RegionOne                            |
| service_id   | 54904230fd73497494018a0964994480     |
| service_name | glance                               |
| service_type | image                                |
| url          | http://openstack-lqlixl.net:9292 |
+--------------+--------------------------------------+

```

#### 2.1.3 创建私有 endpoint： 

```bash
[root@control script]# openstack endpoint create --region RegionOne image internal http://openstack-lqlixl.net:9292
+--------------+--------------------------------------+
| Field        | Value                                |
+--------------+--------------------------------------+
| enabled      | True                                 |
| id           | 95c381550e0b4dc18e368cd589ea7c27     |
| interface    | internal                             |
| region       | RegionOne                            |
| region_id    | RegionOne                            |
| service_id   | 54904230fd73497494018a0964994480     |
| service_name | glance                               |
| service_type | image                                |
| url          | http://openstack-lqlixl.net:9292 |
+--------------+--------------------------------------+

```

#### 2.1.4 ：创建管理 endpoint：

```bash
[root@control script]# openstack endpoint create --region RegionOne image admin http://openstack-lqlixl.net:9292
+--------------+--------------------------------------+
| Field        | Value                                |
+--------------+--------------------------------------+
| enabled      | True                                 |
| id           | 52446fd4e1934ec29c785d190c750269     |
| interface    | admin                                |
| region       | RegionOne                            |
| region_id    | RegionOne                            |
| service_id   | 54904230fd73497494018a0964994480     |
| service_name | glance                               |
| service_type | image                                |
| url          | http://openstack-lqlixl.net:9292 |
+--------------+--------------------------------------+

```

####  2.1.5  验证以上步骤： 

```bash
[root@control script]# openstack endpoint list

```


#### 2.1.6：测试 glance 上传镜像：

```bash
#没有输出是因为glace目前还没有有镜像，是正常的
[root@control script]# glance image-list

```

### 3. 编辑配置文件并同步数据库：

#### 3.1.  编辑配置文件 glance-api.conf：

```bash
vim /etc/glance/glance-api.conf
...
#配置数据库访问：
[database]
connection = mysql+pymysql://glance:redhat@openstack-lqlixl.net/glance
...
#配置认证服务访问：
[keystone_authtoken]
auth_uri = http://openstack-lqlixl.net:5000
auth_url = http://openstack-lqlixl.net:35357
memcached_servers = openstack-lqlixl.net:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = redhat
...
[paste_deploy]
flavor = keystone

#配置本地文件系统存储和镜像文件位置：
[glance_store]
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/


```

#### 3.2 编辑配置文件 glance-registry.conf

```bash

...
#配置数据库访问
[database]
connection = mysql+pymysql://glance:redhat@openstack-lqlixl.net/glance
...

[keystone_authtoken]
auth_uri = http://openstack-lqlixl.net:5000
auth_url = http://openstack-lqlixl.net:35357
memcached_servers = openstack-lqlixl.net:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = redhat
...
[paste_deploy]
flavor = keystone

```



#### 3.3  配置 haproxy 代理 glance

```bash
#修改配置文件
vim /etc/haproxy/haproxy.cfg
...
listen openstack_glance.api_port
 bind 172.20.76.10:9292
 mode tcp
 log global
 server 192.168.1.7  192.168.1.7:9292  check inter 3000 fall 2 rise 5

listen openstack_glance_port
 bind 172.20.76.10:9191
 mode tcp
 log global
 server 192.168.1.7  192.168.1.7:9191  check inter 3000 fall 2 rise 5

#重启服务
systemctl restart haproxy.service

```

#### 3.4 写入镜像服务数据库并验证

```bash
#初始化 glance 数据库：
[root@control ~]# su -s /bin/sh -c "glance-manage db_sync" glance

#验证数据库
mysql -uroot -predhat
show databases;
use glance
show tables;


```

### 4. 启动 glance 并设置为开机启动

```bash
#开启自启服务
systemctl enable openstack-glance-api.service openstack-glance-registry.service
#开启服务
systemctl start openstack-glance-api.service openstack-glance-registry.service

```



### 5.  上传测试验证镜像

- 在 glance 下载一个 0.3.4 版本的 	测试镜像

```bash
#下载镜像
wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img

#执行环境脚本
source admin-ocata.sh

#添加脚本
[root@control ~]# openstack image create "cirros" --file /root/cirros-0.3.4-x86_64-disk.img --disk-format qcow2 --container-format bare --public
+------------------+------------------------------------------------------+
| Field            | Value                                                |
+------------------+------------------------------------------------------+
| checksum         | ee1eca47dc88f4879d8a229cc70a07c6                     |
| container_format | bare                                                 |
| created_at       | 2019-06-21T23:35:29Z                                 |
| disk_format      | qcow2                                                |
| file             | /v2/images/c1315bac-24e7-4f72-ae96-089b1b8d8e7f/file |
| id               | c1315bac-24e7-4f72-ae96-089b1b8d8e7f                 |
| min_disk         | 0                                                    |
| min_ram          | 0                                                    |
| name             | cirros                                               |
| owner            | d92e42680070417a85a05c8478f35913                     |
| protected        | False                                                |
| schema           | /v2/schemas/image                                    |
| size             | 13287936                                             |
| status           | active                                               |
| tags             |                                                      |
| updated_at       | 2019-06-21T23:35:30Z                                 |
| virtual_size     | None                                                 |
| visibility       | public                                               |
+------------------+------------------------------------------------------+

#验证 glance 镜像：
[root@control ~]# glance image-list
+--------------------------------------+--------+
| ID                                   | Name   |
+--------------------------------------+--------+
| c1315bac-24e7-4f72-ae96-089b1b8d8e7f | cirros |
+--------------------------------------+--------+



[root@control ~]# openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| c1315bac-24e7-4f72-ae96-089b1b8d8e7f | cirros | active |
+--------------------------------------+--------+--------+

#查看指定镜像信息：

[root@control ~]# openstack image show cirros
+------------------+------------------------------------------------------+
| Field            | Value                                                |
+------------------+------------------------------------------------------+
| checksum         | ee1eca47dc88f4879d8a229cc70a07c6                     |
| container_format | bare                                                 |
| created_at       | 2019-06-21T23:35:29Z                                 |
| disk_format      | qcow2                                                |
| file             | /v2/images/c1315bac-24e7-4f72-ae96-089b1b8d8e7f/file |
| id               | c1315bac-24e7-4f72-ae96-089b1b8d8e7f                 |
| min_disk         | 0                                                    |
| min_ram          | 0                                                    |
| name             | cirros                                               |
| owner            | d92e42680070417a85a05c8478f35913                     |
| protected        | False                                                |
| schema           | /v2/schemas/image                                    |
| size             | 13287936                                             |
| status           | active                                               |
| tags             |                                                      |
| updated_at       | 2019-06-21T23:35:30Z                                 |
| virtual_size     | None                                                 |
| visibility       | public                                               |
+------------------+------------------------------------------------------+

```




