---
title: jmeter测试并发
author: ninuxGithub
layout: post
date: 2018-11-8 15:35:42
description: "-jmeter测试并发"
tag: jemeter
---
    
    jemter 启动
    1,在TestPlan 上右键创建一个Thread Group 路径： add->Threads->Thread Group
    2,双击Thread Group 修改并发的一些参数；  
    Number of Threads : 100
    Ramp-up period :10
    loop count  : 1
    3.创建一个http请求，thread group 上右键 路径： add->sampler->Http Request
    填写protocal  ip  port number
    填写method  path encoding
    post 需要填写请求参数
    4.在test plan 上右键创建一个 view result : add -> listener->view results in table
    
    
    
    生成报告的命令：
    jmeter -n -t c:\test.jmx -l result.jtl -e -o c:\ResultReport
    
    
    调用test.jmx 配置文件生成一个测试报告在ResutlReport目录
 
 <a href="/images/posts/Test.jmx"> Test.jmx</a>
    
    
    
    
 
    
    
    