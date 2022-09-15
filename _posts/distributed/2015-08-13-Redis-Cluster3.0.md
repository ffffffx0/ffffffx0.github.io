---
layout: post
title: "Redis-Cluster初探"
category : redis
tags : [distributed, redis3.x]
---


网上看着搭建一套redis集群好麻烦，搜了一圈blog,发现还是官网靠谱[Redis cluster tutorial](http://redis.io/topics/cluster-tutorial)

先安装ruby（ruby安装我还是比较烦的，最开始因为ruby安装复杂，且jekyll需要本地编译环境而放弃了

不过这次用的rvm似乎很顺利就搞定了，[jekyll的本地环境配置及主题更换(rvm管理ruby)](http://2pc.github.io/2015/08/05/rvm-ruby-jekyll/)）

解压修改配置文件redis.conf,然后依次拷贝到其他节点，注意修改端口

```java
port 7000
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
```

此时启动redis

```java
src/redis-server ./redis.conf
```

会出现如下日志

```java
18447:M 13 Aug 07:57:37.148 * Increased maximum number of open files to 10032 (it was originally set to 1024).
18447:M 13 Aug 07:57:37.148 * No cluster configuration found, I'm c8b35a4356445746a9855d384b1c2111eacded8c
                _._                                                  
           _.-``__ ''-._                                             
      _.-``    `.  `_.  ''-._           Redis 3.0.3 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._                                   
 (    '      ,       .-`  | `,    )     Running in cluster mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 10002
 |    `-._   `._    /     _.-'    |     PID: 18447
  `-._    `-._  `-./  _.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |           http://redis.io        
  `-._    `-._`-.__.-'_.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |                                  
  `-._    `-._`-.__.-'_.-'    _.-'                                   
      `-._    `-.__.-'    _.-'                                       
          `-._        _.-'                                           
              `-.__.-'                                               

18447:M 13 Aug 07:57:37.158 # Server started, Redis version 3.0.3
18447:M 13 Aug 07:57:37.158 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
18447:M 13 Aug 07:57:37.158 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
18447:M 13 Aug 07:57:37.158 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
18447:M 13 Aug 07:57:37.158 * The server is now ready to accept connections on port 10002
```

建立集群

```java
# src/redis-trib.rb create --replicas 1 172.16.82.186:10001 172.16.82.186:10002 172.16.82.187:10005 172.16.82.187:10006 172.16.82.188:10003 172.16.82.188:10004
>>> Creating cluster
Connecting to node 172.16.82.186:10001: OK
Connecting to node 172.16.82.186:10002: OK
Connecting to node 172.16.82.187:10005: OK
Connecting to node 172.16.82.187:10006: OK
Connecting to node 172.16.82.188:10003: OK
Connecting to node 172.16.82.188:10004: OK
>>> Performing hash slots allocation on 6 nodes...
Using 3 masters:
172.16.82.186:10001
172.16.82.187:10005
172.16.82.188:10003
Adding replica 172.16.82.187:10006 to 172.16.82.186:10001
Adding replica 172.16.82.186:10002 to 172.16.82.187:10005
Adding replica 172.16.82.188:10004 to 172.16.82.188:10003
M: ea91dcad05e9d5b1ff46586a5cc7380133bdc72d 172.16.82.186:10001
   slots:0-5460 (5461 slots) master
S: c8b35a4356445746a9855d384b1c2111eacded8c 172.16.82.186:10002
   replicates 0fdc7426449ce43124b8da11f8d55e430dffaeeb
M: 0fdc7426449ce43124b8da11f8d55e430dffaeeb 172.16.82.187:10005
   slots:5461-10922 (5462 slots) master
S: 52018afc6822a0fe37b873d1ac1502c8c4eaf576 172.16.82.187:10006
   replicates ea91dcad05e9d5b1ff46586a5cc7380133bdc72d
M: 1fa25db9c20217a1adacdcd6705747ec58daac21 172.16.82.188:10003
   slots:10923-16383 (5461 slots) master
S: da09b9f765300ab64915685c2e5570da97d28813 172.16.82.188:10004
   replicates 1fa25db9c20217a1adacdcd6705747ec58daac21
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join...
>>> Performing Cluster Check (using node 172.16.82.186:10001)
M: ea91dcad05e9d5b1ff46586a5cc7380133bdc72d 172.16.82.186:10001
   slots:0-5460 (5461 slots) master
M: c8b35a4356445746a9855d384b1c2111eacded8c 172.16.82.186:10002
   slots: (0 slots) master
   replicates 0fdc7426449ce43124b8da11f8d55e430dffaeeb
M: 0fdc7426449ce43124b8da11f8d55e430dffaeeb 172.16.82.187:10005
   slots:5461-10922 (5462 slots) master
M: 52018afc6822a0fe37b873d1ac1502c8c4eaf576 172.16.82.187:10006
   slots: (0 slots) master
   replicates ea91dcad05e9d5b1ff46586a5cc7380133bdc72d
M: 1fa25db9c20217a1adacdcd6705747ec58daac21 172.16.82.188:10003
   slots:10923-16383 (5461 slots) master
M: da09b9f765300ab64915685c2e5570da97d28813 172.16.82.188:10004
   slots: (0 slots) master
   replicates 1fa25db9c20217a1adacdcd6705747ec58daac21
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

命令行下各种操作还不熟悉，试了下jedis3.x差不多

```java
Set<HostAndPort> jedisClusterNodes = new HashSet<HostAndPort>();  
jedisClusterNodes.add(new HostAndPort("172.16.82.186", 10001));  
jedisClusterNodes.add(new HostAndPort("172.16.82.186", 10002));  
jedisClusterNodes.add(new HostAndPort("172.16.82.187", 10005));  
jedisClusterNodes.add(new HostAndPort("172.16.82.187", 10006));  
jedisClusterNodes.add(new HostAndPort("172.16.82.188", 10003));  
jedisClusterNodes.add(new HostAndPort("172.16.82.188", 10004));  
JedisCluster jc = new JedisCluster(jedisClusterNodes);  
Random r = new Random(10000000);
System.out.println(new String(jc.get("k".getBytes())));
for (int i = 0; i < 1000000; i++) {
	String k = r.nextLong()+""+i;
	System.out.println(k);
	String v = r.nextLong()+""+i;
	jc.set(k, v);
	}
System.out.println(jc.del("*".getBytes()));
System.out.println("====");
```

###  Codis 环境

#### Codis 2.x

#### 源码安装(预先准备好go环境[go学习笔记](http://2pc.github.io/2015/06/12/golang))

```
go get -u -d github.com/CodisLabs/codis
```

#### 默认配置config.ini，修改下zookeeper地址端口，dashboard_addr

```
zk=127.0.0.1:2181
dashboard_addr=172.17.32.127:18087
```

#### 启动 dashboard 

```
bin/codis-config dashboard &
```

#### 初始化 slots

```
bin/codis-config slot init 
```

#### 启动 Codis Redis

```
/data/dev/GoProj/src/github.com/CodisLabs/codis/bin
cp ../test/redis.temp redis.6379.conf 
cp ../test/redis.temp redis.6380.conf 
cp ../test/redis.temp redis.6479.conf 
cp ../test/redis.temp redis.6480.conf 
./codis-server  redis.6379.conf  &
./codis-server  redis.6380.conf & 
./codis-server  redis.6479.conf & 
./codis-server  redis.6480.conf & 
```

####  添加一个group，group的id为1， 并添加一个redis master、slave到该group

```
bin/codis-config server add 1 localhost:6379 master
bin/codis-config server add 1 localhost:6380 slave
bin/codis-config server add 2 localhost:6479 master
bin/codis-config server add 2 localhost:6480 slave
```

#### 设置 server group 服务的 slot 范围

```
bin/codis-config slot range-set 0 511 1 online
bin/codis-config slot range-set 512 1023 2 online
```

#### 启动 codis-proxy

```
bin/codis-proxy -c config.ini -L ./log/proxy.log  --cpu=8 --addr=0.0.0.0:19000 --http-addr=0.0.0.0:11000 & 
```

#### 刚启动的 codis-proxy 默认是处于 offline状态的, 然后设置 proxy 为 online 状态, 只有处于 online 状态的 proxy 才会对外提供服务

```
bin/codis-config -c config.ini proxy online proxy_1

```

#### dashboard

地址： http://172.17.32.127:18087/admin/

#### Codis 3.x

##### 下载源码编译

```
/data/dev/GoProj/src/github.com/CodisLabs/
mv codis codis2.x
https://github.com/CodisLabs/codis/archive/3.0.3.zip
unzip 3.0.3.zip
mv codis-3.0.3/ codis
cd codis
make
```

##### 编译完生成的bin目录

```
total 60484
drwxr-xr-x. 4 root root     4096 Sep 21 11:21 assets
-rwxr-xr-x. 1 root root 14518868 Sep 21 11:21 codis-admin
-rwxr-xr-x. 1 root root 15765944 Sep 21 11:21 codis-dashboard
-rwxr-xr-x. 1 root root  8900440 Sep 21 11:21 codis-fe
-rwxr-xr-x. 1 root root  8668003 Sep 21 11:21 codis-ha
-rwxr-xr-x. 1 root root 10158023 Sep 21 11:21 codis-proxy
-rwxr-xr-x. 1 root root  3743354 Sep 21 11:21 codis-server
-rw-r--r--. 1 root root    32216 Sep 21 11:16 redis.6379.conf
-rw-r--r--. 1 root root    32216 Sep 21 11:16 redis.6380.conf
-rw-r--r--. 1 root root    32216 Sep 21 11:16 redis.6479.conf
-rw-r--r--. 1 root root    32216 Sep 21 11:16 redis.6480.conf
-rw-r--r--. 1 root root    32216 Sep 21 11:16 redis.temp
-rw-r--r--. 1 root root       96 Sep 21 11:21 version
```

##### 创建conf,logs目录，用来存放配置文件，日志

```
mkdir conf
mkdir logs
```

##### dashboard

生成默认配置项

```
bin/codis-dashboard --default-config | tee conf/dashboard.toml
```

启动或停止dashboard

```
nohup bin/codis-dashboard --ncpu=4 --config=conf/dashboard.toml --log=logs/dashboard.log --log-level=WARN &
bin/codis-admin --dashboard=localhost:18080 --shutdown
```

##### codis-proxy

生成默认配置项

```
bin/codis-proxy --default-config | tee conf/cproxy.toml
```
启动proxy

```
nohup bin/codis-proxy --ncpu=4 --config=conf/proxy.toml --log=logs/proxy.log --log-level=WARN &
```

设置proxy为online状态

```
bin/codis-admin --dashboard=172.17.32.127:18080 --create-proxy -x 172.17.32.127:11080
```

停止proxy

```
bin/codis-admin --proxy=172.17.32.127:11080 --shutdown
```
##### codis fe

生成配置文件

```
bin/codis-admin --dashboard-list --zookeeper=127.0.0.1:2181 | tee conf/codis.json
```

启动codis-fe

```
bin/codis-fe --ncpu=4 --log=logs/fe.log --log-level=WARN --dashboard-list=conf/codis.json --listen=0.0.0.0:8080 &
```

##### codis-ha

```
bin/codis-ha --log=logs/codis/ha.log --log-level=WARN --dashboard=172.17.32.127:18080 &
```


##### codis-admin

添加 Redis Server Group

创建group 

```
bin/codis-admin --dashboard=172.17.32.127:18080 --create-group   --gid=1
bin/codis-admin --dashboard=172.17.32.127:18080 --create-group   --gid=2
```
添加服务器server到group(xx80为从库)

```
bin/codis-admin --dashboard=172.17.32.127:18080  --group-add --gid=1 --addr=172.17.32.127:6379
bin/codis-admin --dashboard=172.17.32.127:18080  --group-add --gid=1 --addr=172.17.32.127:6380
bin/codis-admin --dashboard=172.17.32.127:18080  --group-add --gid=2 --addr=172.17.32.127:6479
bin/codis-admin --dashboard=172.17.32.127:18080  --group-add --gid=2 --addr=172.17.32.127:6480
```

slave(xx80)同步master(xx69)

```
bin/codis-admin --dashboard=172.17.32.127:18080 --sync-action --create --addr=172.17.32.127:6380
bin/codis-admin --dashboard=172.17.32.127:18080 --sync-action --create --addr=172.17.32.127:6480
```

升级slave为master

```
bin/codis-admin --dashboard=172.17.32.127:18080 -promote-server --gid=1 --addr=172.17.32.127:6380
```

初始化slots,设置group的范围

```
bin/codis-admin --dashboard=172.17.32.127:18080 --slot-action --create-range --beg=0 --end=511 --gid=1
bin/codis-admin --dashboard=172.17.32.127:18080 --slot-action --create-range --beg=512 --end=1023 --gid=2
```
ref:

1. [codis 3.0.3安装搭建](http://blog.csdn.net/zengxuewen2045/article/details/51559880)

2. [Codis 使用文档](https://github.com/CodisLabs/codis/blob/master/doc/tutorial_zh.md)

