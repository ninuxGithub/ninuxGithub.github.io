---
title: LCN源代码分析
author: ninuxGithub
layout: post
date: 2018-12-10 18:49:29
description: "LCN源代码分析"
tag: tx
---

## lcn源代码分析
    首先从@TxTransaction 开始分析
    更具以往的经验， @TxTransaction 一定是一个标记下的注解， 让后通过某个aop进行了拦截，执行切面，实现事物的代码
    
    lcn执行的代码分析如下
    1.TransactionAspect 创建一个切面 对所有的方法上面的 @TxTransaction 进行扫描 ，然后执行环绕方法
    2.调用AspectBeforeServiceImpl-->around
    	1.创建事物的服务器TransactionServerFactoryServiceImpl.createTransactionServer
    	2.TxStartTransactionServerImpl.execute方法调用
    		1.txManagerService.createTransactionGroup(groupId);创建事组
    		2.TxRunningTransactionServerImpl#execute 在执行service方法的时候 调用  
    		txManagerService.addTransactionGroup(txGroupId, kid, isHasIsGroup, methodStr);添加事物组
    		3. 在 TxStartTransactionServerImpl.execute发生了异常后 
    		执行int rs = txManagerService.closeTransactionGroup(groupId, state);  
    		发送信息关闭事务组 通过Netty发送信息-->NettyServerServiceImpl 初始化netty的serviceBean   
    		是否提交还是回滚  发送消息到netty
     
    3. TxManagerServiceImpl.closeTransactionGroup  tx-manager 接收netty消息开始关闭事务组
    
    
    4. TxManagerSenderServiceImpl.confirm tx-manager 接收netty消息确认事物 
    
    
    
    回顾lcn的原理
    1.创建事物组
    2.添加事务组
    3.关闭事务组
        
    TxCoreServerHandler#channelRead 一直会从channel里面读取消息 调用不同的方法（createTransactionGroup，addTransactionGroup ， closeTransactionGroup）
    tx-client 在执行createTransactionGroup，addTransactionGroup ， closeTransactionGroup     的时候都会发送消息到
    tx-manager的createTransactionGroup，addTransactionGroup ， closeTransactionGroup对应的方法
    
    
    
    
    
    

