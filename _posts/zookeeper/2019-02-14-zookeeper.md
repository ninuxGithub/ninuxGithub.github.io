---
title: zookeeper使用
author: ninuxGithub
layout: post
date: 2019-2-14 17:13:04
description: "Spring cloud sleuth"
tag: zookeeper
---


## zookeeper 的个人理解
    
    zookeeper是一个比较好的开发软件， 可以用于dubbo,hadoop...等项目的配置配置中心，服务的注册中心。 
    
    那么为什么zk可以作为服务的注册中心呢？
    
    zk的特点：
    zk目录结构类似一个倒树形结构， 树木上面可以建立分支， 每个分支可以成为node 节点（自己取名称）
    个人感觉一个非常重要的地方是的  zk 节点的建立 修改 删除 都是原子操作的  这个非常重要
    我们不可能同时建立同名的树节点，这样的话每个节点都是唯一的， 无论是对注册  还是 配置 来说都是很重要的，
    在分布式的项目中 我们可以读到一个唯一的  节点 获取相应的值  （原子操作）；



## zookeeper 入门 
    zookeeper 的学习无非是 目录的建立 修改 节点数据的读取  这些是服务注册和发现的基础
    
    zookeeper master 选举
    zookeeper 分布式队列
    zookeeper 分布式锁
    zookeeper 分布式命名服务 ：  例如为分布式的hibernate 项目提供一个id生成器服务
    
    https://segmentfault.com/a/1190000012185322
    
    这个博客不错 ， 代码都敲了一遍 
    代码寄托在github : https://github.com/ninuxGithub/spring-boot-dubbo-zookeeper.git
    
    
    