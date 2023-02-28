---
title: Github 项目发布到Maven仓库形成dependency
author: ninuxGithub
layout: post
date: 2023-2-27 13:29:13
description: "Github Deploy dependency to Maven repository "
tag: js
---

### 创建Token
    -->github 右上角自己的头像悬浮
    -->点击Settings
    -->滑到最低下点击Developer settings
    -->Personal access tokens
    -->选择Tokens (classic) 创建一个token(最好将所有的权限都勾选上) , 假设为: My_Github_Token
    
    我自己的github 账号为: ninuxGithub
    


### 创建一个github一个公共的仓库
    创建一个仓库， 建立项目，构建自己的项目， clone 到本地

### 开始将项目发布到maven中央仓库
1.配置用户空间下的.m2/settings.xml, 
参考github [官方文档](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-apache-maven-registry)


```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
    <localRepository>D:/apache-maven-calcite</localRepository>
    <servers>
        <server>
            <id>github</id>
            <username>ninuxGithub</username>
            <password>My_Github_Token</password>
            
            <!--使用这个curl 验证token 是否正常-->
            <!--curl -u ninuxGithub:My_Github_Token https://api.github.com/user-->
        </server>
    </servers>
    <profiles>
        <profile>
            <id>github</id>
            <repositories>
                <repository>
                    <id>central</id>
                    <url>https://repo1.maven.org/maven2</url>
                </repository>
                <repository>
                    <id>github</id>
                    <url>https://maven.pkg.github.com/ninuxgithub/ninux-github-facility</url>
                    <snapshots>
                        <enabled>true</enabled>
                    </snapshots>
                </repository>
            </repositories>
        </profile>
    </profiles>

    <activeProfiles>
        <activeProfile>github</activeProfile>
    </activeProfiles>
</settings>

```


2.配置项目的pom.xml
    
    https://maven.pkg.github.com 是官方文档指定的域名
    url 中的地址必须小写： 所以是ninuxgithub (我的用户名称为ninuxGithub)
    ninux-github-facility 是我仓库repository 的名称


```xml
<project>
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>ninux-github-facility</artifactId>
    <version>1.0.0</version>
    <name>ninux-github-facility</name>
    <description>Demo project for Spring Boot</description>
    <packaging>jar</packaging>
    <!--其他的配置忽略-->
    <distributionManagement>
        <repository>
            <id>github</id>
            <name>GitHub ninuxgithub Apache Maven Packages</name>
            <url>https://maven.pkg.github.com/ninuxgithub/ninux-github-facility</url>
        </repository>
    </distributionManagement>
</project>
```


3.开始将项目发布到中央仓库
mvn deploy 

github 项目： https://github.com/ninuxGithub/ninux-github-facility


    




    

    

    

 
    
    
    