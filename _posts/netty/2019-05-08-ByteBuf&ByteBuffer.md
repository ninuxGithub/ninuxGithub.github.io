---
title: ByteBuf vs ByteBuffer 简单的对比
author: ninuxGithub
layout: post
date: 2019-5-8 17:12:11
description: "ByteBuf vs ByteBuffer"
tag: netty
---

### ByteBuffer
    jdk nio 提供了ByteBuffer 来处理io流中的数据，将数据读到或者写入到缓冲区，该类扮演了重要的角色
    
    
    简单的理解：
        ByteBuffer 的父类Buffer 有三个重要的属性 position , limit , capacity
        buffer 可以理解为一个有限的容量的格子容器 ， 每个格子都装了一个byte
        position: 当前读到或写到了那个格子
        limit: 读/写时候意义不同
        capacity：整个容量的大小
        
        
        写的时候capacity 代码整个容器的大小， position 从0开始 往后移动， limit = capacity 
        读的时候capacity 代码整个容器的大小，position 从0开始 往后移动， 
        （考虑到容器未满） 所以limit 为容器装载元素的个数 <=capacity;
        
        flip() : 读写切换；  假如capacity 为10 ，读了5个byte ， position 指向下个位置就是5的位置（0~9)limit =10  = capacity
        调用flip之后position到达0的角标位置 ， limit = 5  ， capacity=10
        
        可以理解为状态的重置了 开启了读模式
        
        ByteBuffer 不会自动的扩容，达到capacity + 1 的时候 抛异常


```java
public class ByteBufferTest {

    public static void main(String[] args) {
        ByteBuffer buffer = ByteBuffer.allocate(32);
        for (int i=0; i<100; i++){
            buffer.put("a".getBytes());

            //当超过buffer声明的长度的时候 ，会抛出异常
            //jdk 源代码
            /*if (length > remaining())
                throw new BufferOverflowException();*/
        }

        //读写切换  切换poistion 的位置  让limit = position 开始从0 读读到limit
        buffer.flip();
        System.out.println(buffer.array());

    }
}
```        



### ByteBuf
    该类来自netty 为了更好的处理管道中的数据
    内部提供了自动扩容，内部维护了 witeableIndex  readableIndex capacity 
    来控制读写；
    
    大概的用法如下 声明ByteBuf有好几种方式


```java
package com.example.springbootstudy.test;

import io.netty.buffer.ByteBuf;
import io.netty.buffer.ByteBufUtil;
import io.netty.buffer.Unpooled;

import java.nio.ByteBuffer;

/**
 * @author shenzm
 * @date 2019-5-8
 * @description 作用
 */
public class ByteBufTest {

    public static void main(String[] args) {
        String str = "aa";
        System.out.println(str.length());
        ByteBuf buffer = Unpooled.wrappedBuffer(str.getBytes());
        System.out.println(new String(buffer.array()));

        //声明大小为8  写入超过8个字节的时候会自动扩容
        ByteBuf bf = Unpooled.buffer(8);
        System.out.println(bf);
        bf.writeBytes("测试a测试b测试".getBytes());
        System.out.println(bf);
        //写的角标
        int widx = bf.writableBytes();

        //读的角标
        int ridx = bf.readableBytes();


        //        ridx     widx
        //|--------|--------|-------|

        System.out.println("ridx :"+ ridx + "  widx: "+ widx);
        byte[] bytes = new byte[widx];
        for (int i = 0; i < ridx; i++) {
            bytes[i] = (bf.getByte(i));
        }
        System.out.println(bf);
        System.out.println(new String(bytes));


        //byteBuf 转 ByteBuffer
        ByteBuffer nioBuffer = bf.nioBuffer();
        System.out.println(new String(nioBuffer.array()));


        System.out.println("==========类型测试================");
        //heapByteBuf 堆内存： 内存的分配和回收快； 缺点对socket io 读写的时候需复制到到内核的缓冲区

        //directByteBuf 直接内存： 非堆内存 相比于堆内存而言分配内存较慢；  比非堆少一次复制 


        ByteBuf heapBuffer = Unpooled.buffer();
        System.out.println(heapBuffer);

        ByteBuf directBuffer = Unpooled.directBuffer();
        System.out.println(directBuffer);

        ByteBuf wrapBuffer = Unpooled.wrappedBuffer("aa".getBytes());
        System.out.println(wrapBuffer);

        ByteBuf copiedBuffer = Unpooled.copiedBuffer("aa".getBytes());
        System.out.println(copiedBuffer);

        System.out.println("==========类型测试================");


        //创建ByteBuf 但是不可以调用array()方法
        ByteBuf byteBuf = ByteBufUtil.threadLocalDirectBuffer();
        byteBuf = byteBuf.writeBytes("java".getBytes());
        //System.out.println(new String(byteBuf.array()));
        System.out.println(byteBuf);

        //refter : https://blog.csdn.net/alex_bean/article/details/51251015


    }
}

```    