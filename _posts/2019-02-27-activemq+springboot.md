---
title: activemq 使用
author: ninuxGithub
layout: post
date: 2019-2-27 17:47:23
description: "activemq 使用"
tag: redis
---


### 消费端配置
    
    因为项目使用了dubbo ,所以配置比较多

```xml
<!--activemq依赖-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-activemq</artifactId>
</dependency>
<dependency>
    <groupId>org.apache.activemq</groupId>
    <artifactId>activemq-pool</artifactId>
    <version>5.14.5</version>
</dependency>
```

```yaml
spring:
  application:
    name: dubbo-provider
    version: 1.0
  activemq:
    broker-url: tcp://10.1.51.96:61616 # activemq broker Url config
    user: admin
    password: admin
    pool:
      enabled: true
      max-connections: 10
    queue-name: test.msg.queue
    out-queue-name: test.msg.out-queue
    topic-name: test.msg.topic
server:
  port: 9090


```



```java
package com.example.provider.config;

import org.apache.activemq.ActiveMQConnectionFactory;
import org.apache.activemq.command.ActiveMQQueue;
import org.apache.activemq.command.ActiveMQTopic;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.jms.annotation.EnableJms;
import org.springframework.jms.config.DefaultJmsListenerContainerFactory;
import org.springframework.jms.config.JmsListenerContainerFactory;

import javax.jms.Queue;
import javax.jms.Topic;

/**
 * @author shenzm
 * @date 2019-2-27
 * @description 作用
 */

@Configuration
@EnableJms
public class ActivemqConfig {

    @Value("${spring.activemq.queue-name}")
    private String queueName;

    @Value("${spring.activemq.topic-name}")
    private String topicName;

    @Value("${spring.activemq.user}")
    private String userName;

    @Value("${spring.activemq.password}")
    private String password;

    @Value("${spring.activemq.broker-url}")
    private String brokerUrl;

    @Bean
    public Queue queue(){
        return new ActiveMQQueue(queueName);
    }

    @Bean
    public Topic topic(){
        return new ActiveMQTopic(topicName);
    }

    @Bean
    public ActiveMQConnectionFactory connectionFactory() {
        return new ActiveMQConnectionFactory(userName, password, brokerUrl);
    }

    @Bean
    public JmsListenerContainerFactory<?> jmsListenerContainerQueue(ActiveMQConnectionFactory connectionFactory){
        DefaultJmsListenerContainerFactory bean = new DefaultJmsListenerContainerFactory();
        bean.setConnectionFactory(connectionFactory);
        return bean;
    }

    @Bean
    public JmsListenerContainerFactory<?> jmsListenerContainerTopic(ActiveMQConnectionFactory connectionFactory){
        DefaultJmsListenerContainerFactory bean = new DefaultJmsListenerContainerFactory();
        //设置为发布订阅方式, 默认情况下使用的生产消费者方式
        bean.setPubSubDomain(true);
        bean.setConnectionFactory(connectionFactory);
        bean.setRecoveryInterval(1000L);
        return bean;
    }
}


package com.example.provider.config;

/**
 * @author shenzm
 * @date 2019-2-27
 * @description 作用
 */

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.jms.annotation.JmsListener;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.stereotype.Component;

@Component
public class QueueListener {
    private static final Logger logger = LoggerFactory.getLogger(QueueListener.class);


    @JmsListener(destination = "${spring.activemq.queue-name}", containerFactory = "jmsListenerContainerQueue")
    @SendTo("${spring.activemq.out-queue-name}")
    public String receive(String text){
        logger.info("QueueListener: consumer-a 收到一条信息: " + text);
        return "consumer-a received : " + text;
    }
}


package com.example.provider.config;

/**
 * @author shenzm
 * @date 2019-2-27
 * @description 作用
 */

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.jms.annotation.JmsListener;
import org.springframework.stereotype.Component;

@Component
public class TopicListener {

    private static final Logger logger = LoggerFactory.getLogger(QueueListener.class);

    @JmsListener(destination = "${spring.activemq.topic-name}", containerFactory = "jmsListenerContainerTopic")
    public void receive(String text){
        logger.info("QueueListener: consumer-a 收到一条信息: " + text);
    }
}

```


### 生产端配置

    pom.xml , application.yml, 差不多 + ActivemqConfig (一样的)
    
```java
package com.example.consumer.controller;


import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jms.annotation.JmsListener;
import org.springframework.jms.core.JmsMessagingTemplate;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import javax.jms.Queue;
import javax.jms.Topic;

@RestController
@RequestMapping("/publish")
public class PublishController {

    @Autowired
    private JmsMessagingTemplate jms;

    @Autowired
    private Queue queue;

    @Autowired
    private Topic topic;

    @RequestMapping("/queue")
    public String queue() {

        for (int i = 0; i < 2; i++) {
            jms.convertAndSend(queue, "queue" + i);
        }

        return "queue 发送成功";
    }

    @JmsListener(destination = "${spring.activemq.out-queue-name}")
    public void consumerMsg(String msg) {
        System.out.println(msg);
    }

    @RequestMapping("/topic")
    public String topic() {

        for (int i = 0; i < 2; i++) {
            jms.convertAndSend(topic, "topic" + i);
        }

        return "topic 发送成功";
    }
}

```    


### 测试
    http://localhost:9091/publish/topic 发送topic 消息， 在消费端有topic 消息收到
    http://localhost:9091/publish/queue 发送的是queue消息， 并且会给生产端一个反馈的消息（相当于一个confrim, 消息确认）


### 项目代码
    https://github.com/ninuxGithub/spring-boot-dubbo-zookeeper
    

### reference
    https://www.cnblogs.com/elvinle/p/8457596.html
    https://www.cnblogs.com/Alex-zqzy/p/9558857.html

     