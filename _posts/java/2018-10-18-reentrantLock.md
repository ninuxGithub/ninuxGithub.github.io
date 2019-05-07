---
title: ReentrantLock vs Synchronized
author: ninuxGithub
layout: post
date: 2018-10-18 08:47:00
description: "可重入锁的ReentrantLock vs Synchronized"
tag: java
---
    
    
    可重入锁的ReentrantLock vs Synchronized
    
## 实现原理
    Synchronized: jdk 的关键字， 锁的功能是jdk对改关键字实现了特殊的功能. 
    ReentrantLock: 出自jdk , 作者doug lea . 是代码的业务逻辑实现的锁的功能， 底层是基于AQS.
    
    区别：Synchronized是编译器去保证加锁和释放锁， 
    而ReentrantLock 需要调用者手动加锁释放锁， 一般需要在fianllly 代码块加入释放锁。
    从锁的颗粒度来说： 后者优于前者


## ReentrantLock的特点
    1.可以指定公平锁还是非公平锁。 而synchronized是非公平的锁。
    公平的定义： 先等待的线程先获取锁， 可以理解为线程排队在一个队列里面， 陆续的获取锁。
    2.ReentrantLock 可以结合使用Condition, 实现分组唤醒线程。 
    sysnchronized 需要结合wait notify notifyAll 来实现线程的通信。
    3.ReentrantLock 提供了中断等待锁的机制， lockInterruptibily 来实现。
    
    lock vs lockInterruptibily的区别
    lock: 轮询的获取锁
    lockInterruptibily: 如果无法获取到锁 则抛出异常给上层调用者。

