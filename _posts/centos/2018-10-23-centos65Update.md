---
title: centos6.5升级内核
author: ninuxGithub
layout: post
date: 2018-10-23 11:21:04
description: "centos 6.5 升级内核"
tag: centos
---
### 1.centos升级
1.查看内核

    uname -r
2.导入公钥数字证书

    rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
3.安装elrepo

    rpm -Uvh https://www.elrepo.org/elrepo-release-6-8.el6.elrepo.noarch.rpm
    
    另外需要安装一个探索最快镜像的包
    yum --enablerepo=elrepo-kernel install kernel-lt -y
    
4.安装kernel长期版本

    #lt表示long-term的意思，长期维护版本，也可以将kernel-lt改为kernel-ml，安装主线版本
    
    $ yum --enablerepo=elrepo-kernel install kernel-lt -y
    
5.修改grub.conf

    vim /etc/grub.conf
    
    # 以下是/etc/grub.conf的内容
    default=0        # 修改该值即可
    timeout=5
    splashimage=(hd0,0)/grub/splash.xpm.gz
    hiddenmenu
    title CentOS (3.10.103-1.el6.elrepo.x86_64)
       
  
6.重启，内核已更新  

    4.4.162-1.el6.elrepo.x86_64

### 2.安装docker
来自：https://blog.csdn.net/sun1021873926/article/details/78007307
1.禁用selinux

    vim /etc/selinux/config的内容
    
    # 以下是/etc/selinux/config的内容
    #     enforcing - SELinux security policy is enforced.
    #     permissive - SELinux prints warnings instead of enforcing.
    #     disabled - No SELinux policy is loaded.
    SELINUX=disabled  # 将SELINUX设为disabled，注意修改后最好重启下机器。

2.安装fedora epel

    $ yum -y install http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm

3.安装docker

    yum install -y docker-io
    
4.以守护线程启动docker

    docker -d
    
5.启动报错

    docker: relocation error: docker: symbol dm_task_get_info_with_deferred_remove, version Base not defined in file libdevmapper.so.1.02 with link time reference
    INFO[0000] Listening for HTTP on unix (/var/run/docker.sock)
    
    # 执行以下内容：
    
    $ yum upgrade device-mapper-libs
    
    
    
    依然报错：FATA[0000] Error mounting devices cgroup: mountpoint for devices not found 
    解决办法：http://www.linuxmysql.com/25/2016/228.htm
    vi /etc/fstab
    在最后一行加入
    none       /sys/fs/cgroup        cgroup        defaults    0   0
    reboot
    
    
6.将docker加入启动运行

    chkconfig docker on
    reboot
    结束
    
    
    
    
        
    


    
    