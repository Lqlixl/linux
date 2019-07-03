#  部署认证服务 keystone:

## 1. keystone简介：

- Keystone 中主要涉及到如下几个概念：User、Tenant、Role、Token:
  - User：使用 openstack 的用户。
  - Tenant：租户,可以理解为一个人、项目或者组织拥有的资源的合集。在一个租户中可以拥有很多个用户，这些用户可以根据权限的划分使用租户中的资源。
  - Role：角色，用于分配操作的权限。角色可以被指定给用户，使得该用户获得角色对应的操作权限。
  - Token：指的是一串比特值或者字符串，用来作为访问资源的记号。Token 中含有可访问资源的范围和有效时间。

## 2. 安装配置



### 2.1 先决条件

 ```bash
#在MySQL创建数据库并授权keystone对keystone仓库的权限
 mysql -uroot -predhat

create database keystone;

grant all privileges on keystone.* to keystone@'%' identified by 'redhat';


#测试控制端登录
yum install mariadb -y

#数据库IP
mysql -ukeystone -predhat -h192.168.1.17
#VIP连接数据库
mysql -ukeystone -predhat -h192.168.1.10

#在控制端设置host
vim /etc/hosts
192.168.1.10 openstack.lqlixl.net

#设置域名方便修改管理
#在使用VIP域名连接
mysql -ukeystone -predhat -hopenstack.lqlixl.com
 ```

### 2.2 部署及配置keystone

#### 2.2.1 安装 keystone：

```bash
#openstack-keystone 是 keystone 服务，http 是 web 服务，mod_wsgi 是 python 的通用网关

yum install -y openstack-keystone httpd mod_wsgi python-memcached
```

#### 2.2.2 编辑 keystone 配置文件：



```bash
#生成临时 token
[root@control ~]# openssl rand -hex 10
0ecdc5d784cbd1e4c954

#修改配置文件
[root@control ~]# vim /etc/keystone/keystone.conf
...
#配置数据库访问
[database]
connection = mysql+pymysql://keystone:redhat@openstack.lqlixl.com/keystone
...
#配置Fernet UUID令牌的提供
[token]
provider = fernet
...


```

#### 2.2.3 初始化并验证数据库

```bash
[root@control ~]# vim /etc/keystone/keystone.conf
[DEFAULT]
admin_token = 0ecdc5d784cbd1e4c954 #手动生成的token
初始化数据库,没有管理员账户密码
解决办法
绕过登录,第一步,直接去第二步



# 会在数据库创建默认表等操作
su -s /bin/sh -c "keystone-manage db_sync" keystone

#keystone日志文件
 ll /var/log/keystone/keystone.log
 
#初始化证书并认证
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone

生成两个key文件，密钥文件
[root@control ~]# ll /etc/keystone/fernet-keys/
total 8
-rw------- 1 keystone keystone 44 Jun 21 11:53 0
-rw------- 1 keystone keystone 44 Jun 21 11:53 1

```

#### 2.2.4 配置 keystone：

- 通过 apache 代理 python：

  ```bash
  
  #软连接配置文件：
  ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
  
  #启动 apache：
  systemctl start httpd && systemctl enable httpd
  
  ```
  
- 查看端口

```bash
ss -ntl
[root@control ~]# ss -ntl
State      Recv-Q Send-Q        Local Address:Port                       Peer Address:Port              
LISTEN     0      128                       *:22                                    *:*                  
LISTEN     0      100               127.0.0.1:25                                    *:*                  
LISTEN     0      511                      :::5000                                 :::*                  
LISTEN     0      511                      :::80                                   :::*                  
LISTEN     0      128                      :::22                                   :::*                  
LISTEN     0      100                     ::1:25                                   :::*                  
LISTEN     0      511                      :::35357                                :::*    

#5000
#35357

```



### 2.3 创建域、用户、项目和角色

#### 2.3.1 通过 admin 的 token 设置环境变量进行操作

- 三个变量设置了，就有管理员权限

```bash
#声明环境变量,直接生成token
export OS_TOKEN=01fe318d6428a091cb13
export OS_URL=http://192.168.1.7:35357/v3
export OS_IDENTITY_API_VERSION=3
```

#### 2.3.2 创建默认域 和项目

- 一定要在上一步设置完成环境变量的前提下方可操作成功，否则会提示未认证。
- 命令格式为：openstack domain create --description "描述信息" 域名

```bash
#各服务器安装openstack 客服端
yum install -y python-openstackclient


 openstack domain create --description "Default Domain" default

#创建 admin 用户并设置密码为 redhat：

openstack user create --domain default --password-prompt admin

User Password:redhat
Repeat User Password:redhat
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | 8999a9638c0749f98d15dda10b402354 |
| enabled             | True                             |
| id                  | d239b2edce494c21a7b686037b23508d |
| name                | admin                            |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

#查看用户清单
openstack user list
openstack domain list

#删除用户的办法
openstack domain delete "用户清单里的ID"




```

#### 2.3.2.1 创建项目

```bash
#创建项目项目
[root@control ~]# openstack project create --domain default --description "Admin Project" admin
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Admin Project                    |
| domain_id   | 8999a9638c0749f98d15dda10b402354 |
| enabled     | True                             |
| id          | d92e42680070417a85a05c8478f35913 |
| is_domain   | False                            |
| name        | admin                            |
| parent_id   | 8999a9638c0749f98d15dda10b402354 |
+-------------+----------------------------------+
```

#### 2.3.3  创建 admin 角色并授权：

##### 2.3.3.1 创建

- 一个项目里面可以有多个角色，目前角色只能创建在/etc/keystone/policy.json 文件中定义好的角色：

```bash
 #创建
 [root@control ~]#  openstack role create admin
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | None                             |
| id        | 7b3f243f44ff4e40950f70ad016f9e85 |
| name      | admin                            |
+-----------+----------------------------------+



```

##### 2.3.3.2 授权

- 将 admin 用户授予 admin 项目的 admin 角色，即给 admin 项目添加一个用户叫 admin，并将其添加至 admin 角色，角色是权限的一种集合：

```bash
#授权
openstack role add --project admin --user admin admin

```



### 2.4 创建service项目

- 各服务之间与 keystone 进行访问和认证，service 用于给服务创建用户
- 会记录openstack用户登录方式

#### 2.4.1 创建sevice项目

```bash
[root@control ~]# openstack project create --domain default --description "ServiceProject" service
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service                          |
|             | Project                          |
| domain_id   | 8999a9638c0749f98d15dda10b402354 |
| enabled     | True                             |
| id          | 357592bf5f7c46fbbc7976ad19f12fe2 |
| is_domain   | False                            |
| name        | service                          |
| parent_id   | 8999a9638c0749f98d15dda10b402354 |
+-------------+----------------------------------+

```

### 2.5 服务注册：

- 将 keystone 服务地址注册到 openstack

#### 2.5.1 ：创建一个 keystone 认证服务

```bash
#查看当前服务
[root@control ~]#  openstack service list

[root@control ~]# openstack service create --name keystone --description "OpenStack Identity" identity
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack                        |
|             | Identity                         |
| enabled     | True                             |
| id          | f2c612c55a8844c7b9f59eebff84e9ad |
| name        | keystone                         |
| type        | identity                         |
+-------------+----------------------------------+

# 验证服务创建成功没有
[root@control ~]# openstack service list
+----------------------------------+----------+----------+
| ID                               | Name     | Type     |
+----------------------------------+----------+----------+
| f2c612c55a8844c7b9f59eebff84e9ad | keystone | identity |
+----------------------------------+----------+----------+
```

#### 2.5.2： 创建endpoint

- 如果创建错误或多创建了，就要全部删除再重新注册，因为你不知道哪一个是对的哪一个是错的，所以只能全部删除然后重新注册，注册的IP地址写keepalived的VIP，稍后配置haproxy：

```bash
#ping vip查看通不通
[root@control ~]# ping openstack.lqlixl.com

# 公共端点
[root@control ~]# openstack endpoint create --region RegionOne identity public http://openstack.lqlixl.com:5000/v3
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 33d0e24ac71b4d4087dff0d9c21054f4 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | f2c612c55a8844c7b9f59eebff84e9ad |
| service_name | keystone                         |
| service_type | identity                         |
| url          | openstack.lqlixl.com:5000/v3 |
+--------------+----------------------------------+



#私有端点

[root@control ~]# openstack endpoint create --region RegionOne identity internal http://openstack.lqlixl.com:5000/v3
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | c6d7ed1be5ea4b4aa54b29bee557c024 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | f2c612c55a8844c7b9f59eebff84e9ad |
| service_name | keystone                         |
| service_type | identity                         |
| url          | openstack.lqlixl.com:5000/v3 |
+--------------+----------------------------------+


#管理端点

[root@control ~]# openstack endpoint create --region RegionOne identity admin http://openstack.lqlixl.com:35357/v3
+--------------+-----------------------------------+
| Field        | Value                             |
+--------------+-----------------------------------+
| enabled      | True                              |
| id           | 4712441d958b4301a546feee131f5786  |
| interface    | admin                             |
| region       | RegionOne                         |
| region_id    | RegionOne                         |
| service_id   | f2c612c55a8844c7b9f59eebff84e9ad  |
| service_name | keystone                          |
| service_type | identity                          |
| url          | openstack.lqlixl.com:35357/v3 |
+--------------+-----------------------------------+

#验证是否都注册进去
[root@control ~]# openstack endpoint list
+---------------------+-----------+--------------+--------------+---------+-----------+----------------------+
| ID                  | Region    | Service Name | Service Type | Enabled | Interface | URL                  |
+---------------------+-----------+--------------+--------------+---------+-----------+----------------------+
| 33d0e24ac71b4d4087d | RegionOne | keystone     | identity     | True    | public    | openstack-lqlixl-    |
| ff0d9c21054f4       |           |              |              |         |           | vip.net:5000/v3      |
| 4712441d958b4301a54 | RegionOne | keystone     | identity     | True    | admin     | openstack-lqlixl-    |
| 6feee131f5786       |           |              |              |         |           | vip.net:35357/v3     |
| c6d7ed1be5ea4b4aa54 | RegionOne | keystone     | identity     | True    | internal  | openstack-lqlixl-    |
| b29bee557c024       |           |              |              |         |           | vip.net:5000/v3      |
+---------------------+-----------+--------------+--------------+---------+-----------+--------------
```



### 2.6  haproxy配置转发

```bash
#添加配置
root@lb:~# cat /etc/haproxy/haproxy.cfg 
...
listen openstack_keystone_port
 bind 192.168.1.10:5000
 mode tcp
 log global
 server 172.20.76.7  172.20.76.7:5000  check inter 3000 fall 2 rise 5

listen openstack__port
 bind 192.168.1.10:35357
 mode tcp
 log global
 server 172.20.76.7  172.20.76.7:35357  check inter 3000 fall 2 rise 5

#重启haproxy
 systemctl restart haproxy

#查看

root@lb:/var/log# ss -ntl
State             Recv-Q             Send-Q                            Local Address:Port                            Peer Address:Port             
 192.168.1.10:5000                                 0.0.0.0:*                
192.168.1.10:5672                                 0.0.0.0:*                
192.168.1.10:3306                                 0.0.0.0:*                
192.168.1.10:11211                                0.0.0.0:*                
0.0.0.0:9999                                 0.0.0.0:*                
192.168.1.10:35357                                0.0.0.0:*                


```



### 2.7 验证

```bash
#在控制端验证
[root@control ~]#  telnet 192.168.1.10 5000
Trying 192.168.1.10...
Connected to 192.168.1.10.
Escape character is '^]'.

[root@control ~]#  telnet 192.168.1.10 35357
Trying 192.168.1.10...
Connected to 192.168.1.10.
Escape character is '^]'.
```

- 测试 keystone 是否可以做用户验证

- 验证 admin 用户，密码 admin，新打开一个窗口并进行以下操作

```bash
#设置变量
[root@control ~]# export OS_IDENTITY_API_VERSION=3

[root@control ~]#  openstack --os-auth-url http://192.168.1.7:35357/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name admin --os-username admin token issue
Password: 
+------------+-----------------------------------------------------------------------------------------------+
| Field      | Value                                                                                         |
+------------+-----------------------------------------------------------------------------------------------+
| expires    | 2019-06-21T07:54:50+0000                                                                      |
| id         | gAAAAABdDH86VyHCI-ROXzQzsnvFOv1S85OTdU54Vdg5kS-                                               |
|            | jGCGNb1RVulsLqpajwQrS6SYF8gUbzeAAb2z7JKn1lh1yL_jBYhVeWqc6TFC0cS412ArK1ok9dGFW4IUqP2_Bth-      |
|            | PWdfKoDS1Gz-YfE1NxvTNwClWsH0YgyK1a-Bj4g4H_12xlgM                                              |
| project_id | d92e42680070417a85a05c8478f35913                                                              |
| user_id    | d239b2edce494c21a7b686037b23508d                                                              |
+------------+---------------------------------------------------------------------------------------
```



```bash
#使用脚本设置环境变量：
  Admin 用户脚本内容：
[root@control ~]# cat /data/script/admin-ocata.sh 

#!/bin/bash
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=redhat
export OS_AUTH_URL=http://192.168.1.7::35357/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2


#以后使用的就直接执行环境脚本。
chmod a+x admin-ocata.sh
```



- 将VIP 改成域名
- 不然存在单点失败的问题



### 3. 创建demo 项目：

#### 3.6.1 创建demo项目

```bash
[root@control ~]#  openstack project create --domain default --description "Demo Project" demo
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Demo Project                     |
| domain_id   | 8999a9638c0749f98d15dda10b402354 |
| enabled     | True                             |
| id          | 10872e27aef84053ab3dbf443e1a9cf9 |
| is_domain   | False                            |
| name        | demo                             |
| parent_id   | 8999a9638c0749f98d15dda10b402354 |
+-------------+----------------------------------+


```

#### 4.6.2 创建 demo 用户并设置密码为 demo：

```bash
[root@control ~]#  openstack user create --domain default --password-prompt demo
User Password: redhat	
Repeat User Password:redhat
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | 8999a9638c0749f98d15dda10b402354 |
| enabled             | True                             |
| id                  | 429eb192cfae403392dec7708494d2a5 |
| name                | demo                             |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

```

#### 4.6.3 使用脚本设置环境变量

```bash
cd /dada/script
[root@control script]# cat demo-ocata.sh 
#!/bin/bash
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=redhat
export OS_AUTH_URL=http://openstack.lqlixl.com:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2


#给脚本执行权限
[root@control script]# chmod a+x demo-ocata.sh
```



### 4. 创建nova和neutron用户

- 将 nova 用户添加到 service 项目并授予 admin 权限

#### 4.1创建nova 密码用户并设置密码为redhat

```bash
[root@control ~]# openstack user create --domain default --password-prompt nova
User Password:redhat
Repeat User Password:redhat
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | 8999a9638c0749f98d15dda10b402354 |
| enabled             | True                             |
| id                  | 6dad69b6bd294885b5b596121d125818 |
| name                | nova                             |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

```



#### 4.2 创建 neutron 用户

- 创建 neutron 用户并设置密码为 neutron：

```bash
[root@control ~]# openstack user create --domain default --password-prompt neutron
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | 8999a9638c0749f98d15dda10b402354 |
| enabled             | True                             |
| id                  | 82f134dff4e44dc888d16811025cd0ad |
| name                | neutron                          |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

```



#### 4.3 对 nova 和 neutron 用户授权

```bash
[root@control ~]# openstack role add --project service --user nova admin
[root@control ~]# openstack role add --project service --user neutron admin

```



#### 5. 创建用户 glance

- 下一章

























































































































































