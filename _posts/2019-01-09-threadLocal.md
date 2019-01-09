---
title: TransmittableThreadLocal使用
author: ninuxGithub
layout: post
date: 2019-1-9 11:10:45
description: "TransmittableThreadLocal使用"
tag: java
---

添加依赖

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>transmittable-thread-local</artifactId>
    <version>2.2.0</version>
</dependency>
```

```java
package com.example.demo.test;

import com.alibaba.ttl.TransmittableThreadLocal;
import com.alibaba.ttl.threadpool.TtlExecutors;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 * @date 2019-1-9
 * @description 作用 多线程提交任务的时候会threadLocal导致传递的变量失效
 */
public class TransmitTableThreadLocalTest {

    static TransmittableThreadLocal<String> threadLocal = new TransmittableThreadLocal<>();
    static ExecutorService pool = TtlExecutors.getTtlExecutorService(Executors.newFixedThreadPool(5));


    //如果采用jdk的threadLocal 难以保证在线程池执行的时候变量的传递
    //static ThreadLocal<String> threadLocal = new InheritableThreadLocal<>();
    //static ThreadLocal<String> threadLocal = new ThreadLocal<>();
    //static ExecutorService pool = Executors.newFixedThreadPool(2);

    public static void main(String[] args) {


        for (int i = 0; i < 100; i++) {
            int index = i;
            pool.execute(new Thread(new Runnable() {

                @Override
                public void run() {

                    //传递值
                    threadLocal.set("java " + index);

                    pool.execute(new Thread(
                            new Runnable() {

                                @Override
                                public void run() {
                                    //获取值
                                    System.out.println(Thread.currentThread().getName() + " 获取修改后的值:" + threadLocal.get());
                                }
                            }
                    ));
                }
            }));
        }

        try {
            Thread.sleep(1000);
            pool.shutdown();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }


    }
}

```


        