---
title: "redis集群高可用"
description: 
date: 2024-01-15T15:56:26+08:00
image: Logo-redis.svg.png
math: 
license: 
hidden: false
comments: true
draft: false
categories:
    - redis
tags:
    - redis
    - 高可用
---
#### 安装
##### 下载源码包&编译

```bash
wget https://download.redis.io/redis-7.2.3.tar.gz
tar -zxf redis-7.2.3.tar.gz
cd redis-7.2.3
make
```

##### 修改配置redis.conf
```
#禁用保护模式，生产勿用！
protected-mode no
```

#### 主从模式
![image.png](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202401151435857.png)

##### 主节点：
``` bash
./src/redis-server /data/redis-master1/redis.conf
```

##### 从节点：
- 修改redis.conf中端口相关及主从复制主节点配置
	``` conf
	port 6380
	# replicaof 主节点IP 主节点端口 replicaof 127.0.0.1 6379
	replicaof 127.0.0.1 6379 
	```
	也可以直接在从节点客户端执行命令：
	``` bash
    redis> replicaof 127.0.0.1 6379
	```
	启动从节点后，可在日志输出中看到主从同步成功。
	![image.png](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202401151236726.png)

##### 验证
主节点：
``` bash
set test this is a key
```
从节点：
``` bash
./src/redis-cli -p 6380

127.0.0.1:6380> get test
```
![image.png](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202401151253772.png)

#### 哨兵模式
![image.png](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202401151507768.png)

##### 配置哨兵节点
配置sentinel.conf文件,其他哨兵也参考该配置进行修改
``` conf
# sentinel节点端口号
port 26379

# sentinel monitor 被监控主节点名称 主节点IP 主节点端口 quorum
sentinel monitor mymaster 127.0.0.1 6379 2

# sentinel down-after-milliseconds 被监控主节点名称 毫秒数
sentinel down-after-milliseconds mymaster 60000

# sentinel failover-timeout 被监控主节点名称 毫秒数
sentinel failover-timeout mymaster 180000

```

##### 启动哨兵节点
``` bash
[root@localhost redis-sentinal1]# ./src/redis-sentinel /data/redis-sentinal1/sentinel.conf
```

其余的哨兵节点也类似。

##### 验证
将master 6379 kill掉，在哨兵节点的日志中可查看到选举主节点的信息：
![image.png](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202401151515725.png)


#### cluster集群模式
Redis Cluster 功能 ： **负载均衡，故障切换**，**主从复制** 。

配置节点：
``` conf
# cluster节点端口号,从6700-6705
port 6700 
# 开启集群模式
cluster-enabled yes 
# 节点超时时间 
cluster-node-timeout 15000
```
分别启动各个节点：
``` bash
./src/redis-server /data/redis-cluster1/redis.conf
```
创建cluster
``` bash
./src/redis-cli -p 6700 --cluster create 127.0.0.1:6700 127.0.0.1:6701 127.0.0.1:6702 127.0.0.1:6703 127.0.0.1:6704 127.0.0.1:6705 --cluster-replicas 1
```
创建成功后会显示集群和节点信息。
![image.png](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202401151546513.png)
也可通过redis客户端执行下面命令，查看集群信息
```
cluster info
```
![image.png](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202401151548911.png)

##### 错误异常解决
如果在配置集群时，启动出现 `[ERR] Node 127.0.0.1:6700 is not empty. Either the node already knows other nodes (check with CLUSTER NODES) or contains some key in database 0.`。这是由于节点中存在以前旧数据导致。

解决方式：
- 停掉所有节点
 - 删除各节点的nodes.conf 和dump.rdb文件
 - 重新启动所有节点
 - 重新创建集群