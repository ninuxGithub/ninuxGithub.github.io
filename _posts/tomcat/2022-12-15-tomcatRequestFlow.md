---
title: Tomcat Request Flow Analysis
author: ninuxGithub
layout: post
date: 2022-12-15 17:36:29
description: "tomcat request"
tag: tomcat
---

### tomcat 的结构是怎样的？
    spring boot 在onRefresh 的时候通过webServerFactory 创建了容器， 然后启动， 其实没有真正启动容器， 只是进行了protocolHandler 的初始化
    当执行finishRefresh 的时候通过调用lifeCycle 来完成 protocolHandler的start, 启动NioEndpoint 

    对tocmcat 结构的描述： https://blog.csdn.net/weixin_40599109/article/details/107720950

![tomcat 结构](/images/posts/tomcat-structure.png)    

### tomcat 是怎么接受请求和将将请求向后传递的

    tomcat 通NioEndpoint 提供服务， startInternal 会创建好一个线程池， 创建一个Poller 轮询线程来发现是否有事件发生， 
    然后开启一个Acceptor线程来接收socket,
    当有客户端连接tomcat 的时候会获取到客户端的连接SocketChannel, 然后通过endpoint.setSocketOptions 方法来设置一下属性， 将socket 包装为socketWrapper
    在将socketWrapper包装为PoolerEvent 添加到Poller 线程的events 集合
    在Poller线程在不断轮询的过程中判断是否有注册的事件， 如果有那么通过processSocket 来处理事件， 创建一个SocketProcessor交给线程池。
    Http11NioProtocol.process(parseReqeuestHeader)----> service ----> getAdapter().service(request, response)--->
    在CoyateAdapter会将coyate.Request转为connector.Request
    然后通过： connector.getService().getContainer().getPipeline().getFirst().invoke(request, response);
    将请求交给pipleLine 的第一个， 依次向后传递
    备注： connector.getService().getContainer()  返回的就是Egine

    * ---Engine
    * ---------Host
    * -------------Context
    * --------------------Wrapper
    * ---------------------------<FilterChain>
    * ---------------------------------------Servlet
    * ----------------------------------------------DispatcherServlet （Spring MVC)
    * ----------------------------------------------Interceptor executtion chain
    * ----------------------------------------------HandlerAdaptor.handle reqeust 
    
    传递请求传递到这些组件的Valve chain

![img.png](/images/posts/tomcat-run-data-line.png)
    