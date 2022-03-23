---
title: 多线程有序执行
author: ninuxGithub
layout: post
date: 2022-3-23 18:35:30
description: "多线程通过condition来有序的执行"
tag: java
---



## example


```java
package com.example.study.test;

import lombok.SneakyThrows;

import java.io.IOException;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

/**
 * 打乱线程的顺序， 线程编号来按照属性执行
 * @author thunderstorm.shen on 2022/3/23
 */
public class SequenceTask {

    static ReentrantLock lock = new ReentrantLock();
    static Condition condition = lock.newCondition();
    static AtomicInteger counter = new AtomicInteger();

    public static void main(String[] args) throws IOException {


        new Thread(new Task(lock, counter, 5, condition), "task-" + 5).start();
        new Thread(new Task(lock, counter, 0, condition), "task-" + 0).start();
        new Thread(new Task(lock, counter, 2, condition), "task-" + 2).start();
        new Thread(new Task(lock, counter, 1, condition), "task-" + 1).start();
        new Thread(new Task(lock, counter, 4, condition), "task-" + 4).start();
        new Thread(new Task(lock, counter, 3, condition), "task-" + 3).start();


    }

    private static class Task implements Runnable {

        private ReentrantLock lock;

        private AtomicInteger counter;

        private int currentNum;

        private Condition condition;

        public Task(ReentrantLock lock, AtomicInteger counter, int currentNum, Condition condition) {
            this.lock = lock;
            this.counter = counter;
            this.currentNum = currentNum;
            this.condition = condition;
        }

        @SneakyThrows
        @Override
        public void run() {
            String name = Thread.currentThread().getName();
            try {
                lock.lock();
                while (currentNum != counter.get()) {
                    System.out.println(name + "进入await, value is : " + counter.get());
                    condition.await();
                }
                System.out.println(currentNum + " is finished..."+ name);
                counter.incrementAndGet();
                condition.signalAll();

            } finally {
                lock.unlock();
            }
        }
    }
}

```

