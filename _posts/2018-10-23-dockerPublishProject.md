---
title: docer部署springboot项目
author: ninuxGithub
layout: post
date: 2018-10-23 14:10:13
description: "docer部署springboot项目"
tag: docker
---
  
## 配置
    项目配置参考https://gitee.com/ninuxGithub/tomcat-nginx-balance.git
    docker发布项目：
    https://www.cnblogs.com/ityouknow/p/8599093.html 
    https://www.jianshu.com/p/efd70ad53602
    
    安装docker,maven,jdk,git 
    maven仓库配置为阿里云的仓库比较快
    开始发布
    mvn package docker:build
    
    docker images: 显示多有的镜像
    docker rmi -f id :删除镜像
    
    
    docker/tomcat-nginx-balance 是镜像的名称， 该项目是spring boot 项目 启动的端口是8888  项目的名称为：tomcatNginxBalance
    
    运行镜像
    docker run -p 8080:8888 docker/tomcat-nginx-balance
    
    -p 8080:8888:将docker的项目映射到虚拟机的8080端口
    访问项目地址：http://xxx.xxx.xx.xx:8080/tomcatNginxBalance/index.html
    xxx.xxx.xx.xx：是虚拟机的ip 
    8080:访问虚拟机的8080端口
    tomcatNginxBalance：项目的名称
    启动成功了
    
    docker ps -all
    CONTAINER ID        IMAGE                         COMMAND                CREATED             STATUS              PORTS                    NAMES
    4d2c81db1cb2        docker/tomcat-nginx-balance   "java -Djava.securit   18 seconds ago      Up 17 seconds       0.0.0.0:8080->8888/tcp   trusting_pasteur    
    
    
    
    看到了PORTS 字段有： 0.0.0.0:8080->8888/tcp 标示映射成功了可以对外访问了
    
    
    
    
    很遗憾centos 6.5在升级后无法启动docker
    
    # service docker start
    Starting cgconfig service: Error: cannot mount cpuset to /cgroup/cpuset: Device or resource busy
    /sbin/cgconfigparser; error loading /etc/cgconfig.conf: Cgroup mounting failed
    Failed to parse /etc/cgconfig.conf or /etc/cgconfig.d[失败]
    Starting docker:        [确定]
    
    
    
    后来不知道怎么就好了， 我将4.4.162-1.el6.elrepo.x86_64内核删除了  ~~~~
    
    
    
    
    
    
    
    
    
    
    
    
    
    