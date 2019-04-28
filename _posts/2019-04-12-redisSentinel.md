---
title: spring boot redis sentinel
author: ninuxGithub
layout: post
date: 2019-4-28 09:10:05
description: "spring boot redis sentinel"
tag: redis
---

### redis sentinel 哨兵    
    在项目中使用reids的时候考虑过没有如果我们的redis由于意外的情况下线了，可能导致项目的瘫痪；
    
    redis集群的模式
        主从复制： master拥有多个slave然后， 从服务器想主服务器发起sync命令， 然后主开始bgsave形成快照并且将上层执行的命令写到
        缓冲区， 然后是想从服务器发送快照 ，从载入快照，执行主服务器缓冲区的命令；
        
        
        哨兵模式： 对主从的优化  对每个节点加入了一个哨兵 监控节点的状态 ； 每个哨兵之间也会发送心跳确定对方是否存活 
        单个哨兵对某个节点发起心跳，发现没有响应 那么就主观的任务服务下线了  sdown
        
        多个哨兵对某个节点发起心跳，发现没有响应 那么多个哨兵一致的任务改节点下线了   odown
        
        然后会从存活的节点中通过特定的算法 推选出一个新的master
        
        
        

    哨兵的测试
    启动虚拟机 开启redis 在6379   哨兵26379    master
    启动虚拟机 开启redis 在6380   哨兵26380    slave1
    启动虚拟机 开启redis 在6381   哨兵26381    slave2
    
    redis.conf

```lombok.config
bind 0.0.0.0
protected-mode yes
port 6379
tcp-backlog 511
timeout 0
tcp-keepalive 300
daemonize yes
supervised no
pidfile /var/run/redis_6379.pid
loglevel notice
logfile "/usr/local/rediscluster/log/master.log"
databases 16
always-show-logo yes

#slave 中才有的配置
#slaveof 10.1.51.230 6379
#masterauth redis

save 900 1
save 300 10
save 60 10000

stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir "/usr/local/rediscluster/master"
slave-serve-stale-data yes
slave-read-only yes
repl-diskless-sync no
repl-diskless-sync-delay 5

repl-disable-tcp-nodelay no
# repl-backlog-size 1mb
# repl-backlog-ttl 3600
slave-priority 100
# min-slaves-to-write 3
# min-slaves-max-lag 10
#
# min-slaves-max-lag is set to 10.
# slave-announce-ip 5.5.5.5
# slave-announce-port 1234
requirepass redis
# maxclients 10000
# maxmemory <bytes>
# maxmemory-policy noeviction
# maxmemory-samples 5


```

    sentinel.conf
    
```lombok.config
daemonize yes
protected-mode no
port 26379
sentinel monitor mymaster 10.1.51.230 6379 1
sentinel auth-pass mymaster redis
sentinel config-epoch mymaster 2
sentinel leader-epoch mymaster 2
sentinel failover-timeout mymaster 180000
sentinel down-after-milliseconds mymaster 30000
sentinel parallel-syncs mymaster 2
```


    启动reids(分别建立3个文件夹 master , slave1 , slave2 创建对应的配置到文件夹)
    redis-server redis.conf
    redis-server sentinel.conf --sentinel
    
    
    redis-cli -p 6379 -a redis
    命令：info replication 可以观察到 role : master 有2个slave节点
    
    
    细节描述： 当启动3个节点的时候  redis 会自动的修改我们的conf文件 ， 会追加一下配置文件类确定slave master信息
    请观察 sentinel.conf 文件  会有自动添加的代码
    
    摸索了很久查阅了一下博客才知道的
    
    ps -aux | grep reids 命令查看当前的所有的redis 进程
    [root@hundsun slave2]# ps -aux|grep redis
    Warning: bad syntax, perhaps a bogus '-'? See /usr/share/doc/procps-3.2.8/FAQ
    root       2489  0.6  0.5  49080  9780 ?        Ssl  19:37   0:03 redis-server 0.0.0.0:6379
    root       2495  0.7  0.4  47036  7756 ?        Ssl  19:38   0:04 redis-server *:26379 [sentinel]      
    root       2502  0.7  0.5  49080  9752 ?        Ssl  19:38   0:03 redis-server 0.0.0.0:6380
    root       2508  0.8  0.4  47036  7748 ?        Ssl  19:38   0:04 redis-server *:26380 [sentinel]      
    root       2513  0.6  0.5  49080  9756 ?        Ssl  19:38   0:03 redis-server 0.0.0.0:6381
    root       2519  0.8  0.4  47036  7752 ?        Ssl  19:38   0:04 redis-server *:26381 [sentinel] 
    
    通过kill -9 2489 来杀死master 进程
    
    然后redis哨兵会通过选举来推选出新的master 注意观察log日志：
    2502:S 26 Apr 19:55:10.993 * Connecting to MASTER 10.1.51.230:6379
    2502:S 26 Apr 19:55:10.994 * MASTER <-> SLAVE sync started
    2502:S 26 Apr 19:55:10.994 # Error condition on socket for SYNC: Connection refused
    2502:S 26 Apr 19:55:12.007 * Connecting to MASTER 10.1.51.230:6379
    2502:S 26 Apr 19:55:12.007 * MASTER <-> SLAVE sync started
    2502:S 26 Apr 19:55:12.007 # Error condition on socket for SYNC: Connection refused
    2502:S 26 Apr 19:55:12.509 * SLAVE OF 10.1.51.230:6381 enabled (user request from 'id=6 addr=10.1.51.230:54412 fd=10 name=sentinel-4ef8ccc5-cmd age=994 idle=0 flags=x db=0 sub=0 psub=0 multi=3 qbuf=14 qbuf-free=32754 obl=36 oll=0 omem=0 events=r cmd=exec')
    2502:S 26 Apr 19:55:12.514 # CONFIG REWRITE executed with success.
    2502:S 26 Apr 19:55:13.022 * Connecting to MASTER 10.1.51.230:6381
    2502:S 26 Apr 19:55:13.023 * MASTER <-> SLAVE sync started
    2502:S 26 Apr 19:55:13.023 * Non blocking connect for SYNC fired the event.
    2502:S 26 Apr 19:55:13.025 * Master replied to PING, replication can continue...
    2502:S 26 Apr 19:55:13.029 * Trying a partial resynchronization (request 086dcbfd12a48914cd16b720a8658c556c778c7f:192270).
    2502:S 26 Apr 19:55:13.035 * Successful partial resynchronization with master.
    2502:S 26 Apr 19:55:13.035 # Master replication ID changed to 19fcbb9229c9c45c944166edcc968ac0e78aa551
    2502:S 26 Apr 19:55:13.035 * MASTER <-> SLAVE sync: Master accepted a Partial Resynchronization.
    2502:S 26 Apr 20:08:35.093 * 1 changes in 900 seconds. Saving...
    2502:S 26 Apr 20:08:35.103 * Background saving started by pid 2606
    2606:C 26 Apr 20:08:35.113 * DB saved on disk
    2606:C 26 Apr 20:08:35.115 * RDB: 6 MB of memory used by copy-on-write
    2502:S 26 Apr 20:08:35.214 * Background saving terminated with success
    
    
    
    说明哨兵推选master成功 服务器节点为6381 为master
    
    
    问题：怎么讲master重新设置为 6379端口的节点呢？
    修改 redis.conf sentinel.conf 文件到原始就可以了（因为在启动redis的时候，经过一轮选举 redis的配置会发送变化）
    
    
### spring boot 集成哨兵    
    
    
```yaml
spring:
  redis:
    sentinel:
      master: mymaster
      nodes: 10.1.51.230:26379,10.1.51.230:26380,10.1.51.230:26381
    password: redis
    database: 0
    jedis:
      pool:
        max-active: 8
        max-wait: -1
        max-idle: 8
        min-idle: 0
    timeout: 6000
    host: 10.1.51.230
    port: 6379
```  


    这个配置是关键， 当master进行选举的时候 spring boot 底层会有代码进行自动的修改master节点，不需要我们手动修改
    
    但我们执行     kill -9 2489
    
    spring boot 的日志 
    INFO 7788 --- [1.51.230:26381]] redis.clients.jedis.JedisSentinelPool    : Created JedisPool to master at 10.1.51.230:6381
    
    说明选举成功了！  ~~~
    
    访问http://localhost:9091/testRedis?value=llllllllllllllllllllll
    
    测试成功， 说明reids master 从之前的6379端口顺利的切换到了6381端口 ~~
    
    
    
    
   
    
        
        