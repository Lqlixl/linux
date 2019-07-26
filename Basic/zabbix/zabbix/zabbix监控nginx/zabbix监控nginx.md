# zabbix监控nginx
## nginx服务器端操作
在nginx上开启状态页
```bash
server {
        ...中间省略...
        location /status {
                stub_status;
        }
}
```
测试状态页是否开启
```bash
root@nginx:~# curl 127.0.0.1/status
Active connections: 1 
server accepts handled requests
 5 5 5 
Reading: 0 Writing: 1 Waiting: 0 
```
编写状态页参数获取脚本
```bash
root@nginx:~# vim check_nginx.sh 
#!/bin/bash
# 
host=${2:-'127.0.0.1'}
port=${3:-'80'}
page=${4:-'status'}
info=$(/usr/bin/curl --connect-timeout 5 -s http://${host}:${port}/${page} 2>/dev/null)
code=$(/usr/bin/curl --connect-timeout 5 -o /dev/null -s -w %{http_code} http://${host}:${port}/${page})
proc=$(/usr/bin/pgrep nginx | wc -l)

case "$1" in
  status)
    echo "$code $proc" | awk '{code=$1}{proc=$2}END{if(code == "200" && proc != 0){printf("%d\n",1)}else{printf("%d\n",0)}}'
    ;;
  active)
    echo "$info" | awk '/^Active/{var=$NF}END{if(var~/^[0-9]+$/){printf("%d\n",var)}else{printf("%d\n",0)}}'
    ;;
  reading)
    echo "$info" | awk '/Reading/ {print $2}'
    ;;
  writing)
    echo "$info" | awk '/Writing/ {print $4}'
    ;;
  waiting)
    echo "$info" | awk '/Waiting/ {print $6}'
    ;;
  accepts)
    echo "$info" | awk 'NR==3 {print $1}'
    ;;
  handled)
    echo "$info" | awk 'NR==3 {print $2}'
    ;;
  requests)
    echo "$info" | awk 'NR==3 {print $3}'
    ;;
  restimes)
    echo "$info" | awk 'BEGIN{OFMT="%.3f"} NR==3 {print $4/$3}'
    ;;
  *)
    echo "ZBX_NOTSUPPORTED"
    ;;
esac
```
将脚本存放到/etc/zabbix/zabbix_agentd.d/目录下，并给与执行权限
```bash
root@nginx:~# mv check_nginx.sh  /etc/zabbix/zabbix_agentd.d/
root@nginx:~# chmod +x /etc/zabbix/zabbix_agentd.d/check_nginx.sh 
```
修改配置文件开启自定义监控项
```bash
UserParameter=nginx.status[*],/etc/zabbix/zabbix_agentd.d/check_nginx.sh $1
```
重启zabbix-agent
```bash
root@nginx:~# systemctl restart zabbix-agent
```
## zabbix服务端测试
测试所有的值都可以获取
```bash
root@zabbix:~# zabbix_get -s 192.168.27.11 -p 10050 -k "nginx.status[status]"
1
root@zabbix:~# zabbix_get -s 192.168.27.11 -p 10050 -k "nginx.status[active]"
1
root@zabbix:~# zabbix_get -s 192.168.27.11 -p 10050 -k "nginx.status[reading]"
0
root@zabbix:~# zabbix_get -s 192.168.27.11 -p 10050 -k "nginx.status[writing]"
1
root@zabbix:~# zabbix_get -s 192.168.27.11 -p 10050 -k "nginx.status[waiting]"
0
root@zabbix:~# zabbix_get -s 192.168.27.11 -p 10050 -k "nginx.status[accepts]"
77
root@zabbix:~# zabbix_get -s 192.168.27.11 -p 10050 -k "nginx.status[handled]"
79
root@zabbix:~# zabbix_get -s 192.168.27.11 -p 10050 -k "nginx.status[requests]"
81
root@zabbix:~# zabbix_get -s 192.168.27.11 -p 10050 -k "nginx.status[restimes]"
0
```
## 创建模板

