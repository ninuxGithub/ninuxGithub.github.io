---
title: Transaction 事物笔记
author: ninuxGithub
layout: post
date: 2019-2-28 08:58:15
description: "Transaction 事物笔记"
tag: tx
---
    

### 事物   
    事物遵循ACID原则
    
    原子性（atomicity）: 事物要么成功提交， 要么全部不提交，是一个整体。
    
    一致性（consistency）: 事物的执行从一种正确的状态转换为另一种状态
    
    隔离性（isolation）: 在事物正确提交之前， 不允许将该事物对数据的改变提供给其他的事物；
    
    持久性（durability）： 事物正确提交后 ， 数据将被持久化到数据库
    
### 数据库的事物
    read uncommited: 读未提交， 那么可能产生脏读， 读到其他事物没有提交的数据
    
    read commited: 度已提交， 只有一个事物提交完成后， 才可去读， 那么可能出现不可重复的
    
    repeatable read : 重复的，读完后加锁， 但是如果有insert 事物的提交会有幻读
    
    serializable: 串行化的执行事物；
    

### spring 事物的传播和隔离级别
    
    来自java类 ：org.springframework.transaction.TransactionDefinition
    
    PROPAGATION_REQUIRED：Support a current transaction; create a new one if none exists.
    如果没有事物创建一个新的事物， 如果有事物，使用事物执行；
    
    
    PROPAGATION_SUPPORTS：Support a current transaction; execute non-transactionally if none exists
    如果有事物，以事物的方式执行， 如果没有事物 以非事物的方式执行；
    
    PROPAGATION_MANDATORY： Support a current transaction; throw an exception if no current transaction
    如果有实物，以事物的方式执行,如果没有事物抛异常
    
    
    PROPAGATION_REQUIRES_NEW：Create a new transaction, suspending the current transaction if one exists.
    创建一个新的事物， 如果当前存在事物，将当前的事物暂停
    
    
    PROPAGATION_NOT_SUPPORTED：Do not support a current transaction; rather always execute non-transactionally.
    以非事物的方式执行， 如果当前存在事物，那么暂停事物
    
    
    PROPAGATION_NEVER：Do not support a current transaction; throw an exception if a current transaction
    以非实物的方式执行， 如果当前有实物那么抛异常
    
    PROPAGATION_NESTED： Execute within a nested transaction if a current transaction exists，
    behave like {@link #PROPAGATION_REQUIRED} otherwise
    如果当前存在事物，那么以嵌套事物的方式执行， 如果当前没有事物， 那么按照required 的方式执行；
    
    
    ISOLATION_DEFAULT:Use the default isolation level of the underlying datastore
    使用当前数据库的默认的事物隔离级别
    
    
    事物传播行为介绍: 
    @Transactional(propagation=Propagation.REQUIRED) 
    如果有事务, 那么加入事务, 没有的话新建一个(默认情况下)
    @Transactional(propagation=Propagation.NOT_SUPPORTED) 
    容器不为这个方法开启事务
    @Transactional(propagation=Propagation.REQUIRES_NEW) 
    不管是否存在事务,都创建一个新的事务,原来的挂起,新的执行完毕,继续执行老的事务
    @Transactional(propagation=Propagation.MANDATORY) 
    必须在一个已有的事务中执行,否则抛出异常
    @Transactional(propagation=Propagation.NEVER) 
    必须在一个没有的事务中执行,否则抛出异常(与Propagation.MANDATORY相反)
    @Transactional(propagation=Propagation.SUPPORTS) 
    如果其他bean调用这个方法,在其他bean中声明事务,那就用事务.如果其他bean没有声明事务,那就不用事务.
    
    事物超时设置:
    @Transactional(timeout=30) //默认是30秒
    
    事务隔离级别:
    @Transactional(isolation = Isolation.READ_UNCOMMITTED)
    读取未提交数据(会出现脏读, 不可重复读) 基本不使用
    @Transactional(isolation = Isolation.READ_COMMITTED)
    读取已提交数据(会出现不可重复读和幻读)
    @Transactional(isolation = Isolation.REPEATABLE_READ)
    可重复读(会出现幻读)
    @Transactional(isolation = Isolation.SERIALIZABLE)
    串行化
    
    不可重复读重点在于update和delete，而幻读的重点在于insert
    
    参考了：https://www.cnblogs.com/jimmy-muyuan/p/5722708.html