---
title: netty对selectedKeys进行优化
author: ninuxGithub
layout: post
date: 2019-9-29 15:40:35
description: "netty之selectedKeys"
tag: netty
---

## 分析netty NioEventLoop 里面的selectedKeys优化处理

    在阅读netty源码的时候对selectedKeys一致有个疑问？ 
    select(boolean b); 方法选择的时候是如何将key放到集合selectedKeys里面去的
    经过debug发现在WindowsSelectorImpl.processFDSet方法里面有selectedKeys.add(sk);
    为什么调用了io.netty.channel.nio.SelectedSelectionKeySet#add 方法；
    而且selectedKey 是SelectorImpl 里面的属性， protected Set<SelectionKey> selectedKeys = new HashSet(); 是不是很奇怪，
    为什么调用到了比人的方法里面去了？  
    
    经过分析NioEventLoop里面的selectedKeys属性， 看看是如何进行初始化的之前真的没有思考太多， 这次发现了一个重要的地方
    NioEventLoop.openSelector()
    
    内部采用了AccessController.doPriviledged 回调函数，绕过权限的检查， 
    通过反射来修改了SelectorImpl类里面的selectedKeys属性的类型；
    
    就是说之前是HashSet的实例对象， 现在改成了自己的类即为： SelectedSelectionKeySet (继承了AbstractSet)  不就是一个Set类型吗；
    
    可以通-Dio.netty.noKeySetOptimization=true 禁止选择的key优化
    
    
    
## 绕过权限的用法

    修改自己定义容器里面的map的类型, 将HashMap的属性动态的修改为ConcurrentHashMap
    饶过权限的地方其他和多， 例如Thread类里面的创建threadGroup的时候
    
    
```java
package com.example.study.reflect;

import java.lang.reflect.Field;
import java.security.AccessController;
import java.security.PrivilegedAction;
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * @author shenzm
 * @date 2019-9-29 15:19
 * @since 1.0
 */
public class AccessControllerTest {


    private static MyContainer myContainer = new MyContainer();

    public void printMap() {
        System.out.println(myContainer.getMap().getClass().getName());
        Map<String, String> map = myContainer.getMap();
        System.out.println(map);


        /**
         * 反射动态的修改加载的类里面的属性的类型
         *
         * 将HashMap改为ConcurrentHashMap
         */
        AccessController.doPrivileged(new PrivilegedAction<Object>() {
            @Override
            public Object run() {
                Class<?> clazz = myContainer.getClass();
                try {
                    Field mapField = clazz.getDeclaredField("map");
                    mapField.setAccessible(true);
                    mapField.set(myContainer, new ConcurrentHashMap<>());
                } catch (Exception e) {
                    e.printStackTrace();
                }
                return null;
            }
        });

        myContainer.getMap().put("aa", "aa");

        //如果还调用map.put添加不进去
        map.put("bb", "bb");

        //将MyContainer的属性map修改为成了ConcurrentHashMap类型
        System.out.println(myContainer.getMap().getClass().getName());

        //输出map的内容
        System.out.println(myContainer.getMap());

    }

    public static void main(String[] args) {
        AccessControllerTest act = new AccessControllerTest();
        act.printMap();
    }

    static class MyContainer {

        //私有属性
        private Map<String, String> map = new HashMap<>();

        public Map<String, String> getMap() {
            return map;
        }
    }
}

```        
    