---
title: spring boot 2.x 使用LCN
author: ninuxGithub
layout: post
date: 2018-12-20 18:30:09
description: "spring boot 2.x 使用LCN"
tag: tx
---

### myIsam & InnoDB
    MyIsam : 默认类型，它是基于传统的ISAM类型， 是存储记录和文件的标准方法。 不是事物安全的，不支持外键，
    但是适合执行大量的select, insert (不受到外键的束缚 速度快)
    
    InnoDB: 支持事物安全的引擎， 支持外键， 行锁。 如果有大量的update , insert 建议使用InnoDB类型。



### spring boot 2.x 使用LCN
    lcn 官网目前给出的lcn 版本为4.1.0 ， 如何使用高版本的spring boot 会出现不兼容的现象
    那么解决方案请参考：  https://www.cnblogs.com/sxdcgaq8080/p/9776695.html
    细节略...
    
    
    遇到的问题： 成功启动了spring cloud 项目 运行测试发现在异常发生后无法进行回滚
    刚开始我还以为是lcn修改的有问题 ， 查看lcn的源代码 
    解决方案： 修改database-platform为database-platform: org.hibernate.dialect.MySQL5InnoDBDialect

```yaml
spring:
  application:
    name: data-server
  jpa:
    hibernate:
      ddl-auto: update
      naming:
        physical-strategy: org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl
    show-sql: true
    database-platform: org.hibernate.dialect.MySQL5InnoDBDialect
#    database-platform: org.hibernate.dialect.MySQL5Dialect  默然会创建的表为MyIsam 类型  不支持事物 ， 自然无法回滚
    database: mysql
  datasource:
    url: jdbc:mysql://localhost:3306/testdb?useUnicode=true&characterEncoding=utf-8
    username: root
    password: root
    driver-class-name: com.mysql.jdbc.Driver
    type: com.alibaba.druid.pool.DruidDataSource
```   
     
     
     修改后完毕的解决spring boot 2.x 整合 lcn 的问题~~
     
    

