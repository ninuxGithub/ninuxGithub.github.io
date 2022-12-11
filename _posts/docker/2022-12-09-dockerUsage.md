---
title: docker 使用
author: ninuxGithub
layout: post
date: 2022-12-9 16:56:39
description: "docker 使用记录随笔"
tag: docker
---
## 1.安装docker 
    卸载已安装的docker: yum remove docker  docker-common docker-selinux docker-engine

    安装： yum install docker

    修改host提供给docker插件连接: cat /etc/docker/daemon.json
    
    {
    "registry-mirrors": ["https://hgnrkfvy.mirror.aliyuncs.com"],
    "hosts": ["tcp://0.0.0.0:2375", "unix:///var/run/docker.sock"]
    }

    重新加载配置文件:systemctl daemon-reload
    重启服务 :  systemctl restart docker.service
    检查: curl http://127.0.0.1:2375/info


## idea 安装docker 连接 虚拟机docker
    坑： 发现idea 的 2022.x.x docker 插件版本连接docker后提示 "Cannot obtain deployments"
    解决方案： 使用idea 2021.1.3， docker 插件版本: 221.7628.21

![img.png](/images/posts/docker-version.png)


## 通过maven 插件打镜像   


```xml
<pom>
    <properties>
        <java.version>11</java.version>
        <docker.registry.url>192.168.23.100</docker.registry.url>
        <docker.registry.host>http://${docker.registry.url}:2375</docker.registry.host>
    </properties>
    <build>
        <plugins>
            <plugin>
                <groupId>com.spotify</groupId>
                <artifactId>docker-maven-plugin</artifactId>
                <version>1.2.1</version>
                <configuration>
                    <!--image name: 192.168.23:100:8080/jxls-demo -->
                    <imageName>${docker.registry.url}:8080/${project.artifactId}</imageName>
                    <imageTags>
                        <imageTag>latest</imageTag>
                    </imageTags>
                    <baseImage>openkbs/jdk11-mvn-py3</baseImage>
                    <maintainer>ninuxgithub@gmail.com</maintainer>
                    <workdir>/ROOT</workdir>
                    <cmd>["java", "-version"]</cmd>
                    <entryPoint>["java", "-jar", "${project.build.finalName}.jar"] </entryPoint>
                    <dockerHost>${docker.registry.host}</dockerHost>
                    <resources>
                        <resource>
                            <targetPath>/ROOT</targetPath>
                            <directory>${project.build.directory}</directory>
                            <include>${project.build.finalName}.jar</include>
                        </resource>
                    </resources>
                </configuration>
                <dependencies>
                    <dependency>
                        <groupId>javax.activation</groupId>
                        <artifactId>activation</artifactId>
                        <version>1.1.1</version>
                    </dependency>
                </dependencies>
            </plugin>

        </plugins>
    </build>
</pom>
```

执行： mvn clean package docker:build 


## 开始查看docker 插件里面的镜像

![img.png](/images/posts/docker-config.png)


## 配置容器内映射端口

![img.png](/images/posts/docker-port-mapping.png)

## 访问
    通过http://192.168.23.100:8080/xxx 访问容器里面的应用






    

    
    