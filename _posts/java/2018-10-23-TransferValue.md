---
title: 值转递vs址传递
author: ninuxGithub
layout: post
date: 2018-10-23 09:11:34 
description: "java传递方法传递的是值/址？"
tag: java
---
    
---------------------------------
直接上代码
---------------------------------


```java

/**
 * <pre>
 *
 * 总结：
 *        1.基本数据类型是传递值的
 *        2.引用数据类型是传递地址
 *        3.String 是传递值的
 *        4.StringBuffer 对象是传递址
 * </pre>
 */
public class TransferValue {
    public static void main(String[] args) {
        TransferValue tv = new TransferValue();
        int a = 99;
        tv.test(a); //传递值， 原始的main中的a不会改变值
        System.out.println("a :" + a);

        Obj obj = new Obj(); //引用类型的变量传递地址
        test(obj);
        System.out.println("main 对象： " + obj.a);

        String s = "java";//传递值， 原始的main中的s不会改变值
        test(s);
        System.out.println("test s main :" + s);


        StringBuffer sb = new StringBuffer("java");//引用类型的变量传递地址
        test(sb);
        System.out.println("teset main sb :" + sb.toString());
    }

    private static void test(StringBuffer sb) {
        sb.append("doc");
        System.out.println("test sb :" + sb.toString());

    }

    private static void test(String s) {
        s = s + "doc";
        System.out.println("test s: " + s);
    }

    private static void test(Obj obj) {
        obj.a = ++obj.a;
        System.out.println("对象传递：" + obj.a);
    }

    private void test(int a) {
        a = ++a;
        System.out.println("test a is : " + a);
    }
}

class Obj {
    int a = 99;

}


```
    
    
    