---
title: jvm内存结构
author: ninuxGithub
layout: post
date: 2018-10-24 08:26:16
description: "jvm内存结构"
tag: java
---


### java的内存结构
jvm内存结构主要分为三个部分： 堆内存， 方法区， 栈。    
1.堆中最大的一块是年轻代，老年代。
* 年轻代：
 >分为是三个部分： Eden, from survivor, to survivor默认的比例为 8:1:1
* 老年代

2.方法区存储类信息， 常量， 静态常量， 是线程共享的区域。   
![jmm](/images/posts/jmm.png)

控制参数
-Xms设置堆的最小空间大小。

-Xmx设置堆的最大空间大小。

-XX:NewSize设置新生代最小空间大小。

-XX:MaxNewSize设置新生代最大空间大小。

-XX:PermSize设置永久代最小空间大小。

-XX:MaxPermSize设置永久代最大空间大小。

-Xss设置每个线程的堆栈大小。

 

没有直接设置老年代的参数，但是可以设置堆空间大小和新生代空间大小两个参数来间接控制。
老年代空间大小=堆空间大小-年轻代大空间大小

从更高的一个维度再次来看JVM和系统调用之间的关系


![jmspace](/images/posts/jmspace.png)

from :https://www.cnblogs.com/ityouknow/p/5610232.html






    
    