# redis cluster集群节点维护
集群运行时间长久之后，难免由于硬件故障、网络规划、业务增长等原因对已有的集群进行相应的调整，比如增加redis节点、节点迁移、更换服务器等
***
## 集群维护之动态增加节点
增加Redis节点，需要与之前的Redis节点版本相同、配置一直，然后分别启动两台redis节点。
## redis集群增加节点实现
1.先将新的节点加入集群
```bash
[root@redis1 ~]# redis-trib.rb add-node 192.168.27.20:6379 192.168.27.11:6379           #将6379加入集群
[root@redis1 ~]# redis-trib.rb add-node 192.168.27.20:6380 192.168.27.11:6379           #将6380加入节点
```
2.查看集群状态
```bash
[root@redis1 ~]# redis-trib.rb info 192.168.27.11:6379
192.168.27.15:6379 (7641530f...) -> 0 keys | 5461 slots | 1 slaves.
192.168.27.13:6379 (87d5d927...) -> 0 keys | 5461 slots | 1 slaves.
192.168.27.20:6380 (388498ca...) -> 0 keys | 0 slots | 0 slaves.
192.168.27.12:6379 (3376d3a9...) -> 0 keys | 5462 slots | 1 slaves.
192.168.27.20:6379 (879f1a6c...) -> 0 keys | 0 slots | 0 slaves.
#新增了2个节点但是还么有主从关系
```
3.在新增的slave节点上指定master
```bash
[root@slave4 ~]# redis-cli -p 6380
127.0.0.1:6380> auth 111111
OK
127.0.0.1:6380> CLUSTER NODES       #查看下节点信息
cc3d1d0c9c64bbce0587507f5d4b0e3233691e2f 192.168.27.16:6379@16379 slave 3376d3a9d73316075955fb59afa6e03b525419a9 0 1560487763569 2 connected
388498ca0920022e6d24c574495306a91ad08779 192.168.27.20:6380@16380 myself,master - 0 1560487762000 8 connected
b6406a84234193e958011b21b01c410aed019091 192.168.27.14:6379@16379 slave 87d5d9278ed230d0817ac36e59dd2d39487e878c 0 1560487761000 3 connected
879f1a6cb119e8640752a778a7cb829f065fd400 192.168.27.20:6379@16379 master - 0 1560487765585 0 connected
7641530f649de98c30f4f02cdc42c299f4c339e4 192.168.27.15:6379@16379 master - 0 1560487763000 7 connected 0-5460
58e21837e91adc28f55cd909e22d610ba4d5e18c 192.168.27.11:6379@16379 slave 7641530f649de98c30f4f02cdc42c299f4c339e4 0 1560487762000 7 connected
87d5d9278ed230d0817ac36e59dd2d39487e878c 192.168.27.13:6379@16379 master - 0 1560487761000 3 connected 10923-16383
3376d3a9d73316075955fb59afa6e03b525419a9 192.168.27.12:6379@16379 master - 0 1560487764577 2 connected 5461-10922
127.0.0.1:6380> CLUSTER REPLICATE 879f1a6cb119e8640752a778a7cb829f065fd400
OK
#将slave设置为192.168.27.20:6379的从服务器，需要跟上master的ID
```
4.再次查看集群信息
```bash
[root@redis1 ~]# redis-trib.rb info 192.168.27.11:6379
192.168.27.15:6379 (7641530f...) -> 0 keys | 5461 slots | 1 slaves.
192.168.27.13:6379 (87d5d927...) -> 0 keys | 5461 slots | 1 slaves.
192.168.27.12:6379 (3376d3a9...) -> 0 keys | 5462 slots | 1 slaves.
192.168.27.20:6379 (879f1a6c...) -> 0 keys | 0 slots | 1 slaves.
[OK] 0 keys in 4 masters.
0.00 keys per slot on average.
#主从配置完毕，但新增的节点上没有槽位，需要对其添加槽位
```
5.重新分配槽位
```bash
[root@redis1 ~]# redis-trib.rb reshard 192.168.27.20:6379
>>> Performing Cluster Check (using node 192.168.27.20:6379)
M: 879f1a6cb119e8640752a778a7cb829f065fd400 192.168.27.20:6379
   slots:0-1364,5461-6826,10923-12287 (4096 slots) master
   1 additional replica(s)
S: 388498ca0920022e6d24c574495306a91ad08779 192.168.27.20:6380
   slots: (0 slots) slave
   replicates 879f1a6cb119e8640752a778a7cb829f065fd400
M: 7641530f649de98c30f4f02cdc42c299f4c339e4 192.168.27.15:6379
   slots:1365-5460 (4096 slots) master
   1 additional replica(s)
S: b6406a84234193e958011b21b01c410aed019091 192.168.27.14:6379
   slots: (0 slots) slave
   replicates 87d5d9278ed230d0817ac36e59dd2d39487e878c
M: 3376d3a9d73316075955fb59afa6e03b525419a9 192.168.27.12:6379
   slots:6827-10922 (4096 slots) master
   1 additional replica(s)
M: 87d5d9278ed230d0817ac36e59dd2d39487e878c 192.168.27.13:6379
   slots:12288-16383 (4096 slots) master
   1 additional replica(s)
S: 58e21837e91adc28f55cd909e22d610ba4d5e18c 192.168.27.11:6379
   slots: (0 slots) slave
   replicates 7641530f649de98c30f4f02cdc42c299f4c339e4
S: cc3d1d0c9c64bbce0587507f5d4b0e3233691e2f 192.168.27.16:6379
   slots: (0 slots) slave
   replicates 3376d3a9d73316075955fb59afa6e03b525419a9
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
How many slots do you want to move (from 1 to 16384)?  4096
What is the receiving node ID? 879f1a6cb119e8640752a778a7cb829f065fd400
Please enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1: all
```
6.分配完毕后再次查看集群状态
```bash
[root@redis1 ~]# redis-trib.rb info
[ERR] Wrong number of arguments for specified sub command
[root@redis1 ~]# redis-trib.rb info 192.168.27.11:6379
192.168.27.15:6379 (7641530f...) -> 0 keys | 4096 slots | 1 slaves.
192.168.27.13:6379 (87d5d927...) -> 0 keys | 4096 slots | 1 slaves.
192.168.27.12:6379 (3376d3a9...) -> 0 keys | 4096 slots | 1 slaves.
192.168.27.20:6379 (879f1a6c...) -> 0 keys | 4096 slots | 1 slaves.
[OK] 0 keys in 4 masters.
0.00 keys per slot on average.
```
新的节点怎加完毕，这里需要注意的是在增加节点时集群内部不能有数据，一旦有数据分配槽位将会报错。需要进入redis将内部数据FLUSHALL。
然后执行以下命令对redis进行修复
```bash
redis-trib.rb fix IPADDR PORT
```
修复完毕后再次重新分配槽位

***
## 集群维护之动态删除节点
添加节点的时候是先添加redis节点到集群，然后分配槽位，删除节点的操作与添加节点相反，是先将被删除的redis节点上的槽位迁移到集群中的其他redis节点上，然后如果一个redis节点上的槽位没有被完全迁移，删除改节点的时候会提示有数据且无法删除。
### 删除节点
1.先将要删除的节点的槽位全部移除
```bash
[root@redis1 ~]# redis-trib.rb reshard 192.168.27.20:6379
>>> Performing Cluster Check (using node 192.168.27.20:6379)
M: 879f1a6cb119e8640752a778a7cb829f065fd400 192.168.27.20:6379
   slots:0-1364,5461-6826,10923-12287 (4096 slots) master
   1 additional replica(s)          #此为要删除的节点
S: 388498ca0920022e6d24c574495306a91ad08779 192.168.27.20:6380
   slots: (0 slots) slave
   replicates 879f1a6cb119e8640752a778a7cb829f065fd400
M: 7641530f649de98c30f4f02cdc42c299f4c339e4 192.168.27.15:6379
   slots:1365-5460 (4096 slots) master
   1 additional replica(s)
S: b6406a84234193e958011b21b01c410aed019091 192.168.27.14:6379
   slots: (0 slots) slave
   replicates 87d5d9278ed230d0817ac36e59dd2d39487e878c
M: 3376d3a9d73316075955fb59afa6e03b525419a9 192.168.27.12:6379
   slots:6827-10922 (4096 slots) master
   1 additional replica(s)
M: 87d5d9278ed230d0817ac36e59dd2d39487e878c 192.168.27.13:6379
   slots:12288-16383 (4096 slots) master
   1 additional replica(s)
S: 58e21837e91adc28f55cd909e22d610ba4d5e18c 192.168.27.11:6379
   slots: (0 slots) slave
   replicates 7641530f649de98c30f4f02cdc42c299f4c339e4
S: cc3d1d0c9c64bbce0587507f5d4b0e3233691e2f 192.168.27.16:6379
   slots: (0 slots) slave
   replicates 3376d3a9d73316075955fb59afa6e03b525419a9
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
How many slots do you want to move (from 1 to 16384)? 4096      #需要移动多少个槽位
What is the receiving node ID? 7641530f649de98c30f4f02cdc42c299f4c339e4     #将槽位分配到哪个节点上去
Please enter all the source node IDs.       #槽位从哪里来
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1:879f1a6cb119e8640752a778a7cb829f065fd400   
Source node #2:done         
Do you want to proceed with the proposed reshard plan (yes/no)? yes
#是否确认
```
移除完毕后查看下状态
```bash
[root@redis1 ~]# redis-trib.rb info 192.168.27.11:6379
192.168.27.15:6379 (7641530f...) -> 0 keys | 8192 slots | 2 slaves.
192.168.27.13:6379 (87d5d927...) -> 0 keys | 4096 slots | 1 slaves.
192.168.27.12:6379 (3376d3a9...) -> 0 keys | 4096 slots | 1 slaves.
192.168.27.20:6379 (879f1a6c...) -> 0 keys | 0 slots | 0 slaves.    #已经没有槽位，可以删除了
[OK] 0 keys in 4 masters.
0.00 keys per slot on average.
```
删除节点
```bash
[root@redis1 ~]# redis-trib.rb del-node 192.168.27.11:6379 879f1a6cb119e8640752a778a7cb829f065fd400
>>> Removing node 58e21837e91adc28f55cd909e22d610ba4d5e18c from cluster 192.168.27.11:6379
>>> Sending CLUSTER FORGET messages to the cluster...
>>> SHUTDOWN the node.
#节点就被删除了
```