---
title: ReentrantLock写一个倒计时运行程序
author: ninuxGithub
layout: post
date: 2019-1-8 18:46:45
description: "ReentrantLock写一个倒计时运行程序"
tag: java
---


### 目的
    测试使用ReentrantLock

### 使用描述
    使用ReentrantLock写一个倒计时的程序，倒计时5s 然后开始运行 ， 显示倒计时的剩余时间
    
代码如下：

```java
package com.example.demo.concurrent;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * @author shenzm
 * @date 2019-1-8
 * @description 测试ReentrantLock
 */
public class ReentrantLockTest {


    /**
     * 线程A等待5s 然后线程B唤醒A运行其他的业务代码， 线程C是一个倒计时的线程，作用是倒数秒
     * @param args
     */
    public static void main(String[] args) {
        final int leftSeconds = 5;
        Service service = new Service(leftSeconds);
        service.testLock();
        new Thread(()->{
            String name = Thread.currentThread().getName();
            for(int i=leftSeconds; i>0; i--){
                try {
                    System.out.println(name + " 等待 " + i + "s..");
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            service.signal();
        },"倒计时线程(C)").start();
    }

    static class Service {
        //空的构造器为 unfair  : true 为fail
        private Lock lock = new ReentrantLock();
        private Condition condition = lock.newCondition();

        private volatile int leftSeconds;

        public Service(int leftSeconds) {
            this.leftSeconds = leftSeconds;
        }

        public void testLock() {
            new Thread(() -> {
                try {
                    String name = Thread.currentThread().getName();
                    lock.lock();
                    System.out.println(name + "进入等待 " + leftSeconds-- + "s..");
                    condition.await();
                    System.out.println(name + "处理好多代码....");
                } catch (Exception e) {
                    e.printStackTrace();
                } finally {
                    lock.unlock();
                }
            }, "A").start();
        }

        private void signal() {
            new Thread(() -> {
                try {
                    lock.lock();
                    String name = Thread.currentThread().getName();
                    System.out.println(name + "唤醒等待的线程");
                    condition.signalAll();
                } catch (Exception e) {
                    e.printStackTrace();
                } finally {
                    lock.unlock();
                }
            }, "B").start();
        }

    }

}

```    

结果如下

```text
A进入等待 5s..
倒计时线程(C) 等待 5s..
倒计时线程(C) 等待 4s..
倒计时线程(C) 等待 3s..
倒计时线程(C) 等待 2s..
倒计时线程(C) 等待 1s..
B唤醒等待的线程
A处理好多代码....
```    