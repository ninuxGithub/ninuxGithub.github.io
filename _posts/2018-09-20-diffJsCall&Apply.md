---
title: js apply和call的区别
author: ninuxGithub
layout: post
date: 2018-09-20 10:09:22
description: "js apply和call的区别 "
tag: js
---
    js apply 和call 的却别， 两个的功能其实是一样的，只是用法上面有点区别， 观测代码如下
    apply:传递的参数需要是数组
    call:传递的参数是逗号隔开的
    
```javascript

    function add(c,d){
        return this.a + this.b + c + d;
    }

    var s = {a:1, b:2};
    console.log(add.call(s,3,4)); // 1+2+3+4 = 10
    console.log(add.apply(s,[5,6])); // 1+2+5+6 = 14 


```    
 
    
    
    