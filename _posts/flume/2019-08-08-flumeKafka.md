---
title: Flume 推送日志到Kafka
author: ninuxGithub
layout: post
date: 2019-8-8 16:28:40
description: "Flume 推送日志到Kafka"
tag: flume
---

### 目的
    为了记录正确的方式来进行flume日志推送到kafka；
    
    
#开始实践
    需要
        centos7.x
        准备下载  apache-flume-1.8.0-bin.tar.gz，  kafka_2.12-2.3.0.tgz, 安装zookeeper-3.5.5;
        
    
    安装步骤忽略；
    
    第一步：zookeeper配置
    
    zoo.cfg文件的配置
    
```lombok.config
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/tmp/zookeeper
clientPort=2181
#需要验证
authProvider.1=org.apache.zookeeper.server.auth.SASLAuthenticationProvider
requireClientAuthScheme=sasl
jaasLoginRenew=3600000
admin.serverPort=6666
```    

    zk_server_jaas.conf文件的配置
    
```lombok.config
Server {
    org.apache.kafka.common.security.plain.PlainLoginModule required 
    username="admin" 
    password="admin-2019" 
    user_kafka="kafka-2019" 
    user_producer="prod-2019";
};
```    
    在zkEnv.sh里面修改
    export SERVER_JVMFLAGS="-Xmx${ZK_SERVER_HEAP}m $SERVER_JVMFLAGS  -Djava.security.auth.login.config=/usr/local/apache-zookeeper-3.5.5/conf/zk_server_jaas.conf"
    
    
    然后进行jar包的添加将
    lz4-java-1.6.0.jar, kafka-clients-2.3.0.jar, snappy-java-1.1.7.3.jar
    
    
    
    第二部：配置kafka
    
```lombok.config
broker.id=100
#如果这个地方使用的是localhost 在其他的地方配置需要一样不然会导致无法正常的连接
listeners=SASL_PLAINTEXT://localhost:9092
advertised.listeners=SASL_PLAINTEXT://localhost:9092
security.inter.broker.protocol=SASL_PLAINTEXT  
sasl.enabled.mechanisms=PLAIN  
sasl.mechanism.inter.broker.protocol=PLAIN  
authorizer.class.name=kafka.security.auth.SimpleAclAuthorizer
allow.everyone.if.no.acl.found=true
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
log.dirs=/tmp/kafka-logs
num.partitions=1
num.recovery.threads.per.data.dir=1
offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1
log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000
zookeeper.connect=10.1.51.240:2181
#需要验证增加的超时时间
zookeeper.connection.timeout.ms=6000000
group.initial.rebalance.delay.ms=0
```


    zookeeper.properties 文件的配置
    
```lombok.config
dataDir=/tmp/zookeeper
clientPort=2181
maxClientCnxns=0
authProvider.1=org.apache.zookeeper.server.auth.SASLAuthenticationProvider
requireClientAuthScheme=sasl
jaasLoginRenew=3600000
```    
    
    producer.properties 文件的配置
    
```lombok.config
security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
bootstrap.servers=localhost:9092
compression.type=none
```    

    consumer.properties 文件的配置
    
```lombok.config
security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
zookeeper.connect=10.1.51.240:2181
zookeeper.connection.timeout.ms=6000
group.id=test-consumer-group
```   

    
     kafka_client_jaas.conf 文件的配置
    
```lombok.config
KafkaClient {
        org.apache.kafka.common.security.plain.PlainLoginModule required
        username="producer"
        password="prod-2019";
};
```    


     kafka_server_jaas.conf 文件的配置
    
```lombok.config
KafkaServer {
        org.apache.kafka.common.security.plain.PlainLoginModule required
        username="admin"
        password="admin-2019"
        user_admin="admin-2019"
        user_producer="prod-2019"
        user_consumer="cons-2019";
};
#提供给zookeeper验证
Client {
org.apache.kafka.common.security.plain.PlainLoginModule required
        username="kafka"
        password="kafka-2019";
};
```    



    kafka-server-start.sh修改
    export KAFKA_HEAP_OPTS="-Xmx1G -Xms1G -Djava.security.auth.login.config=/usr/local/kafka_2.12/config/kafka_server_jaas.conf"
 
    kafka-console-producer.sh, kafka-console-consumer.sh 都需要修改 加入kafka_client_jaas.conf
    
    
    
    第三部： flume配置
    
    flume2Kafka.properties文件的配置

```lombok.config
Flume2Kafka.sources=mysource
Flume2Kafka.channels=mychannel
Flume2Kafka.sinks=mysink
Flume2Kafka.sources.mysource.channels=mychannel
Flume2Kafka.sources.mysource.shell = /bin/sh -c
Flume2Kafka.sources.mysource.type=exec
#目标文件
Flume2Kafka.sources.mysource.command=tail -F /opt/project/logs/mine.server/warn.log
Flume2Kafka.sinks.mysink.channel=mychannel
Flume2Kafka.sinks.mysink.type=org.apache.flume.sink.kafka.KafkaSink
#Flume2Kafka.sinks.mysink.type=logger
#kafka服务地址
Flume2Kafka.sinks.mysink.kafka.bootstrap.servers=localhost:9092
#SASL加密配置
Flume2Kafka.sinks.mysink.kafka.producer.security.protocol=SASL_PLAINTEXT
Flume2Kafka.sinks.mysink.kafka.producer.sasl.mechanism=PLAIN
#提交监控数据的topic
Flume2Kafka.sinks.mysink.kafka.topic=monitor-data
Flume2Kafka.sinks.mysink.kafka.serializer=kafka.serializer.StringEncoder
Flume2Kafka.sinks.mysink.kafka.batchSize=20
Flume2Kafka.sinks.mysink.kafka.producer.linger.ms=30
Flume2Kafka.sinks.mysink.kafka.producer.compression.type=snappy
Flume2Kafka.sinks.mysink.kafka.producer.requiredAcks=1
Flume2Kafka.sinks.mysink.kafka.producer.acks=1
Flume2Kafka.channels.mychannel.type=memory
Flume2Kafka.channels.mychannel.capacity=1000
Flume2Kafka.channels.mychannel.transactionCapacity=100
```    


     flume-env.sh文件的配置
     
     
```lombok.config
export JAVA_OPTS="$JAVA_OPTS -Djava.security.auth.login.config=/usr/local/apache-flume-1.8.0/conf/flume_plain_jaas.conf" 
export JAVA_HOME=/usr/local/jdk1.8.0_131
```     
     flume_plain_jaas.conf 文件的配置
     
     
```lombok.config
KafkaClient {
        org.apache.kafka.common.security.plain.PlainLoginModule required
        username="producer"
        password="prod-2019";
};
```     



    细节问题： 
    
    monitor-data 是topic的名称；
    
    创建topic
    bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic monitor-data
    
    查看topic
    bin/kafka-topics.sh --list --zookeeper localhost:2181
    
    删除topic
    bin/kafka-topics.sh --describe --zookeeper localhost:2181 -topic monitor-data
    
    
    开是消息的生产
    bin/kafka-console-producer.sh --broker-list localhost:9092 --topic monitor-data  --producer.conf /usr/local/kafka_2.12/config/producer.properties 
    
    消费开始
    bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic monitor-data --from-beginning --consumer.config /usr/local/kafka_2.12/config/consumer.properties 
    
    启动flume
    sh bin/flume-ng agent -n Flume2Kafka -c conf -f conf/flume2kafka.properties -Dflume.root.logger=INFO,console
    
    启动项目产生日志， 发现日志可以读到kafka并且可以正常的消费了；
    
    参考了
    https://www.2bowl.info/kafka%E7%9A%84saslplain%E8%AE%A4%E8%AF%81%E9%85%8D%E7%BD%AE%E8%AF%B4%E6%98%8E/
    https://www.cnblogs.com/zongzong/p/6566656.html
    https://segmentfault.com/a/1190000012990954
    https://blog.csdn.net/m0_37786447/article/details/80577798

            

            



    
     