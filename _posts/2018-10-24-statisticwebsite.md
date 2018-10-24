---
title: 站点统计
author: ninuxGithub
layout: post
date: 2018-10-24 14:01:58
description: "站点统计"
tag: web
---
 
## 站点统计计数
    采用仆算子
    博客地址：https://www.jianshu.com/p/e3d9f6777fb8

    js加入
    <script async src="//busuanzi.ibruce.info/busuanzi/2.3/busuanzi.pure.mini.js"></script>
    
    html页面为：
    <span id="busuanzi_container_page_pv"> | 阅读：<span id="busuanzi_value_page_pv"></span>次</span>
    
    
## 百度统计
    注册一个百度统计的账号：https://tongji.baidu.com/web/welcome/login
    添加一个想要统计的站点，然后会生成一个id
    
    然后将这个代码放入到页面内, 607730063baee35487f9b4401b19431c 就是我的区分web站点的id
```javascript
    var _hmt = _hmt || [];
    (function() {
      var hm = document.createElement("script");
      hm.src = "https://hm.baidu.com/hm.js?607730063baee35487f9b4401b19431c";
      var s = document.getElementsByTagName("script")[0]; 
      s.parentNode.insertBefore(hm, s);
    })();

```

    怎么判断是否安装统计js成功呢？
    访问被统计的web站点， f12 查看network 里面是否有调用过hm.js?607730063baee35487f9b4401b19431c&xxxx
    如果有就代表成功了~~~~~

    