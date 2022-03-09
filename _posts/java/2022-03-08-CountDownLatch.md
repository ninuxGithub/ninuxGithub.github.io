---
title: CountDownLatch 运行分析
author: ninuxGithub
layout: post
date: 2022-3-8 10:08:16
description: "CountDownLatch 运行分析"
tag: java
---



## example


```java
public class CountDownLatchTest {

    @Test
    public void test() throws InterruptedException {
        int num = 2;
        CountDownLatch latch = new CountDownLatch(num);
        for (int i = 0; i < num; i++) {
            new Thread(() -> {
                System.out.println("线程： " + Thread.currentThread().getName() + " start===>");
                try {
                    TimeUnit.SECONDS.sleep(10);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName() + " execute business code");
                latch.countDown();
            }, "countDown-thread-" + i).start();
        }

        latch.await();
        System.out.println("finished.....");

    }
}
```


## CountDownLatch 运行分析
    测试类参考： CountDownLatchTest

    创建CountDownLatch 干了什么？
    当运行测试类的时候， 开始创建了一个CountDownLatch 传递参数2， 会在内部创建一个Sync(AQS) 设置state = 2;
    countDown 方法会每次进行cas 操作的state - 1 的操作， 当state减下去， 一直到了0 ， 说明 所有的countDown方法都已经执行完毕
    也就是我们想要的目标线程数量2 已经完成所有的任务 （state == 0）
    
    await: 主要使用了就行当前调用线程阻塞的， 那么是否需要阻塞取决于上面state 的值是否 ==0? 1: -1, 
    如果state> 0, 说明目标任务没有完全执行完毕， 那么需要对当前的调用线程就行阻塞， 那么怎么阻塞的呢？ 这个就是AQS内部作用体现出来了
    
    AQS 为什么可以进行等待的作用？
    AQS 你不会维护一个双向的Node(prev, next, waitStatus) 双向链表, 链表上是一个个的Node 节点， 
    Node 有2种类型
    一种是独享的SHARED， 比如CountDownLatch就是共享锁， 多个线程使用同个Node节点来进行阻塞，通过状态判断是否解除阻塞执行后续代码
    另外一种是EXCLUSIVE， 独占锁， 排他的， 所以只会被一个线程使用到， 后续补充
    

    await 方法怎么让当前线程等待的？
    await 首先确定了status >0 才会创建一个Node （N） ， 让后创建一个new Node() 作为head, 将N 挂载到 head 的后面
    head -> N 形成了一个双写链表， 遍历链表， 
    再次判断一次state是否==0， 如果是说明线程执行完毕， 直接break 跳出循环就好了， 不需要阻塞了
    如果state> 0 , 那么通过LockSupport.park(this); 对向前的线程就行阻塞， 那么当前发起 测试的是main线程， 那么main线程就会被阻塞
    
    等到2个runnable 的task 执行完毕后， 怎么让main 苏醒呢？
    countDown 会调用doReleaseShared, 当且仅当state 进行cas 操作 设置为0 的时候才会进行unparkSucessor
 


