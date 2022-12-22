---
title: docer部署springboot项目
author: ninuxGithub
layout: post
date: 2018-10-23 14:10:13
description: "docer部署springboot项目"
tag: docker
---

### 配置
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

### docker命令
    docker images: 查看容器中的镜像
    docker ps -all: 运行的镜像
    docker top id :查看镜像的状态
    docker logs -f id :查看镜像的日志
    docker inspect id ：查看docker底层状态 返回json
    docker stop id : 停止某个镜像
    docker run -p 8080:8888 docker/tomcat-nginx-balance  ： 运行一个镜像（名称为：docker/tomcat-nginx-balance）  启动在虚拟机的 8080端口  ， 部署在docker的8888端口
    docker rm id : 删除一个镜像


### Dockerfile的语法
    from : 设置镜像使用的基础镜像
    maintainer: 设置镜像的维护者
    run:编译镜像的时候运行的脚本
    cmd:设置容器启动命令
    label:设置镜像的标签
    expose:镜像暴露的端口
    env:容器的环境变量
    add:编译的时候复制文件到镜像中
    copy:编译的时候复制文件到镜像中
    entrypoint: 设置容器入口的程序
    volume: 设置容器的挂载卷
    user:设置运行run,cmd,entrypoint的用户名称
    workdir: 设置run，cmd,entrypoint,copy,add指令的工作目录
    arg: 设置镜像时候加入的参数
    onbuild: 设置镜像onbuild指令
    stopsignal: 容器的退出的信号量
    
    
    指令：FROM 
    功能描述：设置基础镜像 
    语法：FROM < image>[:< tag> | @< digest>] 
    提示：镜像都是从一个基础镜像（操作系统或其他镜像）生成，可以在一个Dockerfile中添加多条FROM指令，一次生成多个镜像 
    注意：如果忽略tag选项，会使用latest镜像
    
    指令：MAINTAINER 
    功能描述：设置镜像作者 
    语法：MAINTAINER < name>
    
    指令：RUN 
    功能描述： 
    语法：RUN < command> 
              RUN [“executable”,”param1”,”param2”] 
    提示：RUN指令会生成容器，在容器中执行脚本，容器使用当前镜像，脚本指令完成后，Docker Daemon会将该容器提交为一个中间镜像，供后面的指令使用 
    补充：RUN指令第一种方式为shell方式，使用/bin/sh -c < command>运行脚本，可以在其中使用\将脚本分为多行 
              RUN指令第二种方式为exec方式，镜像中没有/bin/sh或者要使用其他shell时使用该方式，其不会调用shell命令 
    例子：RUN source $HOME/.bashrc;\ 
              echo $HOME
    
              RUN [“/bin/bash”,”-c”,”echo hello”]
    
              RUN [“sh”,”-c”,”echo”,”$HOME”] 使用第二种方式调用shell读取环境变量
    
    指令：CMD 
    功能描述：设置容器的启动命令 
    语法：CMD [“executable”,”param1”,”param2”] 
              CMD [“param1”,”param2”] 
              CMD < command> 
    提示：CMD第一种、第三种方式和RUN类似，第二种方式为ENTRYPOINT参数方式，为entrypoint提供参数列表 
    注意：Dockerfile中只能有一条CMD命令，如果写了多条则最后一条生效
    
    指令：LABEL 
    功能描述：设置镜像的标签 
    延伸：镜像标签可以通过docker inspect查看 
    格式：LABEL < key>=< value> < key>=< value> … 
    提示：不同标签之间通过空格隔开 
    注意：每条指令都会生成一个镜像层，Docker中镜像最多只能有127层，如果超出Docker Daemon就会报错，如LABEL ..=.. <假装这里有个换行> LABEL ..=..合在一起用空格分隔就可以减少镜像层数量，同样，可以使用连接符\将脚本分为多行 
              镜像会继承基础镜像中的标签，如果存在同名标签则会覆盖
    
    指令：EXPOSE 
    功能描述：设置镜像暴露端口，记录容器启动时监听哪些端口 
    语法：EXPOSE < port> < port> … 
    延伸：镜像暴露端口可以通过docker inspect查看 
    提示：容器启动时，Docker Daemon会扫描镜像中暴露的端口，如果加入-P参数，Docker Daemon会把镜像中所有暴露端口导出，并为每个暴露端口分配一个随机的主机端口（暴露端口是容器监听端口，主机端口为外部访问容器的端口） 
    注意：EXPOSE只设置暴露端口并不导出端口，只有启动容器时使用-P/-p才导出端口，这个时候才能通过外部访问容器提供的服务
    
    指令：ENV 
    功能描述：设置镜像中的环境变量 
    语法：ENV < key>=< value>…|< key> < value> 
    注意：环境变量在整个编译周期都有效，第一种方式可设置多个环境变量，第二种方式只设置一个环境变量 
    提示：通过${变量名}或者 $变量名使用变量，使用方式${变量名}时可以用${变量名:-default} ${变量名:+cover}设定默认值或者覆盖值 
              ENV设置的变量值在整个编译过程中总是保持不变的
    
    指令：ADD 
    功能描述：复制文件到镜像中 
    语法：ADD < src>… < dest>|[“< src>”,… “< dest>”] 
    注意：当路径中有空格时，需要使用第二种方式 
              当src为文件或目录时，Docker Daemon会从编译目录寻找这些文件或目录，而dest为镜像中的绝对路径或者相对于WORKDIR的路径 
    提示：src为目录时，复制目录中所有内容，包括文件系统的元数据，但不包括目录本身 
              src为压缩文件，并且压缩方式为gzip,bzip2或xz时，指令会将其解压为目录 
              如果src为文件，则复制文件和元数据 
              如果dest不存在，指令会自动创建dest和缺失的上级目录
    
    指令：COPY 
    功能描述：复制文件到镜像中 
    语法：COPY < src>… < dest>|[“< src>”,… “< dest>”] 
    提示：指令逻辑和ADD十分相似，同样Docker Daemon会从编译目录寻找文件或目录，dest为镜像中的绝对路径或者相对于WORKDIR的路径
    
    指令：ENTRYPOINT 
    功能描述：设置容器的入口程序 
    语法：ENTRYPOINT [“executable”,”param1”,”param2”] 
              ENTRYPOINT command param1 param2（shell方式） 
    提示：入口程序是容器启动时执行的程序，docker run中最后的命令将作为参数传递给入口程序 
              入口程序有两种格式：exec、shell，其中shell使用/bin/sh -c运行入口程序，此时入口程序不能接收信号量 
              当Dockerfile有多条ENTRYPOINT时只有最后的ENTRYPOINT指令生效 
              如果使用脚本作为入口程序，需要保证脚本的最后一个程序能够接收信号量，可以在脚本最后使用exec或gosu启动传入脚本的命令 
    注意：通过shell方式启动入口程序时，会忽略CMD指令和docker run中的参数 
              为了保证容器能够接受docker stop发送的信号量，需要通过exec启动程序；如果没有加入exec命令，则在启动容器时容器会出现两个进程，并且使用docker stop命令容器无法正常退出（无法接受SIGTERM信号），超时后docker stop发送SIGKILL，强制停止容器 
    例子：FROM ubuntu <换行> ENTRYPOINT exec top -b
    
    指令：VOLUME 
    功能描述：设置容器的挂载点 
    语法：VOLUME [“/data”] 
              VOLUME /data1 /data2 
    提示：启动容器时，Docker Daemon会新建挂载点，并用镜像中的数据初始化挂载点，可以将主机目录或数据卷容器挂载到这些挂载点
    
    指令：USER 
    功能描述：设置RUN CMD ENTRYPOINT的用户名或UID 
    语法：USER < name>
    
    指令：WORKDIR 
    功能描述：设置RUN CMD ENTRYPOINT ADD COPY指令的工作目录 
    语法：WORKDIR < Path> 
    提示：如果工作目录不存在，则Docker Daemon会自动创建 
              Dockerfile中多个地方都可以调用WORKDIR，如果后面跟的是相对位置，则会跟在上条WORKDIR指定路径后（如WORKDIR /A   WORKDIR B   WORKDIR C，最终路径为/A/B/C）
    
    指令：ARG 
    功能描述：设置编译变量 
    语法：ARG < name>[=< defaultValue>] 
    注意：ARG从定义它的地方开始生效而不是调用的地方，在ARG之前调用编译变量总为空，在编译镜像时，可以通过docker build –build-arg < var>=< value>设置变量，如果var没有通过ARG定义则Daemon会报错 
              可以使用ENV或ARG设置RUN使用的变量，如果同名则ENV定义的值会覆盖ARG定义的值，与ENV不同，ARG的变量值在编译过程中是可变的，会对比使用编译缓存造成影响（ARG值不同则编译过程也不同） 
    例子：ARG CONT_IMAG_VER <换行> RUN echo $CONT_IMG_VER 
              ARG CONT_IMAG_VER <换行> RUN echo hello 
              当编译时给ARG变量赋值hello，则两个Dockerfile可以使用相同的中间镜像，如果不为hello，则不能使用同一个中间镜像
    
    指令：ONBUILD 
    功能描述：设置自径想的编译钩子指令 
    语法：ONBUILD [INSTRUCTION] 
    提示：从该镜像生成子镜像，在子镜像的编译过程中，首先会执行父镜像中的ONBUILD指令，所有编译指令都可以成为钩子指令
    
    指令：STOPSIGNAL 
    功能描述：设置容器退出时，Docker Daemon向容器发送的信号量 
    语法：STOPSIGNAL signal 
    提示：信号量可以是数字或者信号量的名字，如9或者SIGKILL，信号量的数字说明在Linux系统管理中有简单介绍
    
    
    
    
    
    
    
    
    
    
    
    
    
    