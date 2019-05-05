---
title: centos7 docker使用
author: ninuxGithub
layout: post
date: 2019-5-5 13:55:51
description: "centos7 docker使用"
tag: docker
---
  
## docker 构建
    将项目：https://gitee.com/ninuxGithub/tomcat-nginx-balance.git克隆到centos
    安装好环境： 直接采用命令  mvn package docker:build会报错不知道为什么？
    Cannot retry request with a non-repeatable request entity:xxxxxxxxxx
    那么分开命令构建吧
    首先mvn package
    查看到target地下是有jar生成了
    到target/docker目录
    采用命令  docker build -t tomcat-nginx-balance:v1.0
    改了会构建一个名称为tomcat-nginx-balance的镜像  tag为： v1.0
    [root@localhost docker]# docker images
    REPOSITORY             TAG                 IMAGE ID            CREATED             SIZE
    tomcat-nginx-balance   v1.0                a0080d0ae5f8        9 minutes ago       146MB
    openjdk                8-jdk-alpine        3675b9f543c5        3 weeks ago         105MB
    
    启动tomcat-nginx-balance容器
    docker run -p 8080:8888 tomcat-nginx-balance:v1.0
    一定要加入tag版本  不然无法启动
    
    
    那么为什么之前会通过命令： mvn package docker:build 无法通过呢  ？
    查看了    https://blog.csdn.net/weixin_34414196/article/details/86813424
    
    原来是docker使用的时候artifactId 不能为大写 修改了 
    <docker.image.prefix>tomcatNginxBalance</docker.image.prefix>
    
    改成
    <docker.image.prefix>docker</docker.image.prefix>
    
    
    
    然后采用 ： mvn package docker:build
    [root@localhost tomcat-nginx-balance]# docker images
    REPOSITORY                    TAG                 IMAGE ID            CREATED             SIZE
    docker/tomcat-nginx-balance   latest              98b92ba62734        6 seconds ago       146MB
    openjdk                       8-jdk-alpine        3675b9f543c5        3 weeks ago         105MB
    观测到docker.image.prefix是镜像名称的前缀
    
   
   
   
### intellij项目部署到centos
    需要docker开启以一个tcp的监听
    vim /usr/lib/systemd/system/docker.service
    
    在ExecStart=/usr/bin/dockerd-current 后面加上
    
    -H tcp://0.0.0.0:2375 -H unix://var/run/docker.sock
    
    然后
    systemctl daemon-reload
    systemctl start docker
    
    然后对项目打包 ， 运行docker项目
   
  ![echart docker 项目部署](/images/posts/docker-intellij.png)
  
    启动项目成功后 项目会被部署到centos 
    访问http://10.1.51.240:8080/tomcatNginxBalance/index.html 可以观察到ok部署成功
    项目被成功的通过docker容器启动了，项目会在docker中形成一个镜像文件
    
    docker images 查看所有的镜像
    
    docker ps -a 查启动的项目程序   在port有对外映射的端口信息说明ok了
    
    参考了https://blog.csdn.net/jackcheng1117/article/details/83080303
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
   
    
    
    
    
    
    
    
    
    
    
    
    
    
    