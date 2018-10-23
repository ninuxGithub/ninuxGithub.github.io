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
    
    
    docker/tomcat-nginx-balance 是镜像的名称
    
    运行镜像
    docker run -p 8080:8080 docker/tomcat-nginx-balance
    
    很遗憾centos 6.5在升级后无法启动docker
    
    # service docker start
    Starting cgconfig service: Error: cannot mount cpuset to /cgroup/cpuset: Device or resource busy
    /sbin/cgconfigparser; error loading /etc/cgconfig.conf: Cgroup mounting failed
    Failed to parse /etc/cgconfig.conf or /etc/cgconfig.d[失败]
    Starting docker:        [确定]
    
    
    
    
    
    
    
    
    
    
    
    
    
    