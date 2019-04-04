---
title: 项目远程debug
author: ninuxGithub
layout: post
date: 2019-4-4 09:09:12
description: "项目远程debug"
tag: java
---


### 目的
    项目有时候在上线的情况下会出现一些和本地运行的不一样的结果，让人很诧异，本篇结束远程debug
    
### 准备工作
    虚拟机，安装好tomcat,构建一个spring boot 项目打好war包,包名称为： tomcatNginxBalance.war
    
    修改我们的tomcat 的catalina.sh
    
    加入一行命令：CATALINA_OPTS="-Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=n,address=6666"
    address=6666 可以自定义的修改端口
    
    然后启动tomcat!  远程debug 首先必须要启动远程项目 ~~~~
    
    查看logs/catalina.log
    日志的第一行会有：Listening for transport dt_socket at address: 6666
    
    
    ok 项目启动监听成功 ， 等待远程debug连接
    
    
    在说说百度搜索其他的remote debug教程有些说的不对
    1.远程debug 和root用户启动tomcat 没有关系， 即使是root 启动的tomcat 一样可以进行远程debug
    2.没有必要配置tomcat 的home 环境变量
    
    其他的步骤就大概是一样的了
    
### eclipse 远程debug

    项目上右键 debug as --> debug configurations..  
    新建一个远程debug application:
    
![ecliseDebug](/images/posts/eclipseDebug.jpg)     

    点击debug按钮 ， 加入断点， 请求远程的请求： http://10.1.51.230:8080/tomcatNginxBalance/balance
    
    然后在本地会有debug进入对应的requestmapping
       
```java
class TestController{
   @RequestMapping(value = "/balance")
   	public Map<String, Object> test(HttpServletRequest request) {
   		String ip = WebHelper.retriveClientIp(request);
   		logger.info("访问来自：{}", ip);
   		Map<String, Object> map = new HashMap<>();
   		map.put("ip", ip);
   		return map;
   	} 
}
```    


### intelliJ idea 远程debug

    intellij debug 更加方便
        
![ecliseDebug](/images/posts/intellijDebug.jpg)
   


### 项目地址
    项目的寄托地址：https://gitee.com/ninuxGithub/tomcat-nginx-balance.git  