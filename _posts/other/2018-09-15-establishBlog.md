---
title: 搭建Jekylly博客
author: ninuxGithub
layout: post
date: 2018-09-15 17:13:06 
description: "搭建Jekylly博客的简单步骤"
tag: jeylly
---
    
    今天第一次尝试搭建一个github博客页面，大概的步骤如下
    1.选择一个你喜欢的jekyll模板并且clone到本地。
    2.安装ruby(我是windows版本的)，下载安装包直接安装， 检查： ruby -v 出现版本则为成功。
    3.安装jekyll:  gem install jekyll;  jekyll -v: 出现版本则为成功；
    4.可能需要安装bundler: gem install bundler   
    5.本地启动项目： jekyll server  如果没有报错则启动成功； --> (bundle exec jekyll server)
    
    启动的参数：
    Configuration file: D:/dev/live2/blog/_config.yml
                Source: D:/dev/live2/blog
           Destination: D:/dev/live2/blog/_site
     Incremental build: disabled. Enable with --incremental
          Generating...
                        done in 0.877 seconds.
      Please add the following to your Gemfile to avoid polling for changes:
        gem 'wdm', '>= 0.1.0' if Gem.win_platform?
     Auto-regeneration: enabled for 'D:/dev/live2/blog'
        Server address: http://127.0.0.1:4000/blog/
      Server running... press ctrl-c to stop.
 
    
    
    