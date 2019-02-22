---
title: redis分布式锁
author: ninuxGithub
layout: post
date: 2019-2-20 10:20:07
description: "redis分布式锁"
tag: redis
---

## redis 分布式锁

    原理： 采用jedis 实现的方法jedis.set(String key, String value, String nxxx, String expx, int time)
    第一个为key，我们使用key来当锁，因为key是唯一的。
    
    第二个为value，我们传的是requestId，很多童鞋可能不明白，有key作为锁不就够了吗，为什么还要用到value？
    原因就是我们在上面讲到可靠性时，分布式锁要满足第四个条件解铃还须系铃人，通过给value赋值为requestId，
    我们就知道这把锁是哪个请求加的了，在解锁的时候就可以有依据。requestId可以使用UUID.randomUUID().toString()方法生成。
    
    第三个为nxxx，这个参数我们填的是NX，意思是SET IF NOT EXIST，即当key不存在时，我们进行set操作；若key已经存在，则不做任何操作；
    
    第四个为expx，这个参数我们传的是PX，意思是我们要给这个key加一个过期的设置，具体时间由第五个参数决定。
    
    第五个为time，与第四个参数相呼应，代表key的过期时间。
    
    
    对应的是  redis 的命令 ：  set key value nxxx time   
    解释： 如果key 不存在 就设置key  如果key 存在了 让这个key 的失效时间 expire 有效时间为 time 
    redis 单机 只可以设置一个同名的key 从而达到锁的 机制
    底层的原理是redis watch , watch 这个key  是否存在 具体的watch 了解不多
    
    
### 代码实现
    
```java
package com.example.api.redis;


import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import redis.clients.jedis.Jedis;

import java.util.Collections;
import java.util.UUID;
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

/**
 * @author shenzm
 * @date 2019-2-22
 * @description 作用
 * <p>
 * sleep 不释放锁对象 , 但是让出cpu的执行权
 * <p>
 * wait 释放锁  进入等待池  等待notify唤醒
 */
public class RedisLockUtil {

    private static final Logger logger = LoggerFactory.getLogger(RedisLockUtil.class);

    private static final String LOCK_SUCCESS = "OK";
    private static final String SET_IF_NOT_EXIST = "NX";
    private static final String SET_WITH_EXPIRE_TIME = "PX";
    private static final Long RELEASE_SUCCESS = 1L;

    static ExecutorService threadPool = Executors.newFixedThreadPool(5);


    public static boolean tryGetDistributedLock(Jedis jedis, String lockKey, String requestId, int expireTime) {
        String result = jedis.set(lockKey, requestId, SET_IF_NOT_EXIST, SET_WITH_EXPIRE_TIME, expireTime);
        if (result == null) {
            logger.info("tryLock 的结果为null");
        }
        if (null != result && LOCK_SUCCESS.equals(result)) {
            return true;
        }
        return false;
    }


    public static boolean releaseDistributedLock(Jedis jedis, String lockKey, String requestId) {
        String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
        Object result = jedis.eval(script, Collections.singletonList(lockKey), Collections.singletonList(requestId));
        if (result == null) {
            logger.info("releaseLock 的结果为null");
        }
        if (null != result && RELEASE_SUCCESS.equals(result)) {
            return true;
        }
        return false;

    }

    public static Object exeTask(Jedis jedis, Callable callable, String lockKey) {
        String threadName = Thread.currentThread().getName();
        String requestId = UUID.randomUUID().toString();
        int expireTime = 10_000;
        Object result = null;
        synchronized (RedisLockUtil.class) {
            try {
                boolean tryLock = tryGetDistributedLock(jedis, lockKey, requestId, expireTime);
                while (!tryLock) {
                    try {
                        //如果没有获取到锁 等待别的线程任务执行完毕释放锁
                        Thread.sleep(10);
                    } catch (InterruptedException e) {

                    }
                    tryLock = tryGetDistributedLock(jedis, lockKey, requestId, expireTime);
                }
                if (tryLock) {
                    Future submit = threadPool.submit(callable);
                    try {
                        result = submit.get();
                    } catch (Exception e) {
                        logger.error("{} 线程发生了异常 ： {}", threadName, e);
                    }
                }
            } finally {
                releaseDistributedLock(jedis, lockKey, requestId);
            }
            return result;
        }
    }


}


```      

#### 使用Demo
    开启50个线程调用这个分布式的锁 执行一个线程 返回一个结果

```java
package com.example.api.redis;

import redis.clients.jedis.Jedis;

import java.util.UUID;
import java.util.concurrent.*;

/**
 * @author shenzm
 * @date 2019-2-22
 * @description 作用
 */
public class RedisLockTest {

    static ExecutorService threadPool = Executors.newFixedThreadPool(5);

    public static void main(String[] args) {
        int num = 50;
        CountDownLatch latch = new CountDownLatch(num);
        Jedis jedis = new Jedis("10.1.51.96", 6379);
        jedis.auth("redis");

        for (int i = 0; i <num ; i++) {
            final String threadName = "thread" + i;
            new Thread(new Runnable() {
                @Override
                public void run() {
                    Object taskResult = RedisLockUtil.exeTask(jedis, new Callable<Object>() {
                        @Override
                        public Object call() throws Exception {
                            try {
                                //模拟计算消耗的时间
                                Thread.sleep(1000);
                            } catch (InterruptedException e) {
                                e.printStackTrace();
                            }
                            return Math.random() * 100;
                        }
                    },"lockKey");
                    System.out.println(threadName + "线程任务执行返回的结果： "+taskResult);
                    try {
                        latch.countDown();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            }, threadName).start();

        }
        try {
            latch.await();
            System.out.println("====>计算完毕");
        } catch (Exception e) {
            e.printStackTrace();
        }

    }

}

``` 



### reference 
    参考博客：http://www.cnblogs.com/linjiqin/p/8003838.html
    
    源代码请访问：https://github.com/ninuxGithub/spring-boot-dubbo-zookeeper.git   






