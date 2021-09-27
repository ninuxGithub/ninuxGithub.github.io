---
title: Netty 源码阅读理解
author: ninuxGithub
layout: post
date: 2021-8-8 11:00:29
description: "Netty源码"
tag: netty
---


## Netty源码阅读的感悟
    1。首先要理解刀netty 是采用了Reactor 多线程 mainReactor, 和subReactor的一个基于事件的Nio 框架。
    内部高度的封装好了java nio 的逻辑，不在需要开发者自己去大量的开发，管理channel建立连接， 管理线程任务的处理， 以及
    监听selector 是否有时间发生，然后根据特定的事件来做特定的额业务， Netty 都以及高度集成的封装好了， 那么是怎么玩转的呢？
    

## Reactor 线程模型
    netty 使用的是reactor 线程模型， bookGroup 内部会有一个selector 不断地轮询监听他感兴趣的事件（Accept, Connect）.
    监听到了事件以后会将当前建立连接的channel 注册到另外一个workerGroup Eventloop 里面持有的一个selector 上去， 怎么获取
    里面的EventLoop 呢？ 通过策略（依据当前拥有eventLoop 的池子大小决定， 看看是不是2的幂决定使用那种策略）
    然后获取到subGroup 里面的Selector 然后不断地监听读写事件，调用pipeline 里面的处理器（判断是入栈还是出栈的顺序依次经过处理器，
    例如我们自己定义的Decoder, Encoder, 和业务处理器Handle , 他们都是基于inbound 或者outBound handle 而实现的）
    然后通过类似于一种filter 的方式， 经过每个处理器， 完成自己负责要处理的逻辑， 交个下一个处理器。从而完成了整个消息的读写。


## Netty任务
    netty里面有个taskQueue,和scheduleQueue, 在我们执行业务代码的时候可能会调用channel.eventLoop.execute(), 那么类似这样的动作
    都会被放到整个taskQueue 里面去， 还有一种是定时的schedule的任务， 那么是怎么玩的呢？
    当NioEventLoop里面的线程被调用后（执行了run方法以后） 当前的selector 会不断地去监听读写的事件processSelectedKey动作就是处理当前
    获取到的读写事件， 对其进行响应处理。 在finally 里面会有一个逻辑runAllTask(time) 就是用来处理非io 任务的， 那么当schedule 里面的任务
    到达了执行的时间点的时候， netty会fetchTaskFromSchedule  （字面意思自己理解一下） ， 将可以执行的schedule 任务丢到了 taskQueue
    所以我们后续要关注的就是TaskQueue 不断地消费任务， 执行任务就OK了， 大概思想是这样的。像程序启动注册的registry0方法在第一次执行的时候
    就是通过eventLoop.execute 方法来调用的， 那么此时就会触发了  execute 的调用， execute 的调用又触发了 NioEventLoop 线程的run 方法
    就是开始了监听轮询， 处理事件， 然后将Task 丢到taskQueue里面去（execute不是立刻执行， 我想这也是类似于线程池的思想一样， 拥有一个缓冲的队列
    ，放到线程里面去执行，不至于立刻执行而且任务执行的又慢导致线程的堵死）

## Netty 一些思想
    netty channel 的创建都是基于了反射工厂， 传递一个Class, 然后通过工厂模式来创建的。
    netty 里面的eventLoop , 也是有一个选择的策略， power of two & genera 2种方式
    还有一个就是netty 里面对selectionSelectedKey 集合的优化， 牛叉，相当于在启动的时候， 直接修改了Class 里面对象的类型， 使用了自己的Set
    取代了java Nio 的集合， 从而可以提高了selector 可以选择事数量件可以拓展 (扩容)。

## selector 注册
    selector 在netty 里面扮演了至关重要的就是，别人都说netty 实现了多路复用，个人理解就是这个selector 起了决定重要的作用。
    在windows 里面默认是使用的windowsSelectorImpl,  select 方式来继续事件的监听的。 那么在linux 里面就是基于epoll 的方式来监听事件
    他们后会创建fd, 然后注册fd, 监听事件， 这是select需要轮询所有的fd set ,然后epoll, 会有一个callback ， 内核会主动通知时间在那个fd 上面
    有事件发生了。
        mainReactor 的selector 是通过registry0 方法将channel 注册到selector 上去的， 
    那么当发生了accept/connect (相对与服务的客户端不同的事件而已，都是建立链接) ，然后就会开始将但其的channel 注册到workerReacotr 的selector 
    上去, 那么怎么注册的呢？ 这个因为是我debug 很久才发现的， 在ServerBootstrapAcceptor里面的channelRead 方法里面有一行代码
    childGroup.register(child).addListener， 这个就是 将channel注册到  workerReactor 的selector 上至关重要的一行代码。

## handler & channelPipeline
    上面讲到了 handler 是类似于一种过滤器一样的 ， 一个接这个一个维护在pipeline 里面的（就是双向链表）。
    其实放到pipeline 里面的是 channelHandlerContext 封装了一个EventLoop, name,和当前的handler ， 所以个我们可以在handle 里面获取一个‘
    eventLoop 的东西去执行execute


~~ 大概就这么多吧， 后续如果有想到在更新
    




      

