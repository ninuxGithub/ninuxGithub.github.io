---
title: ThreadLocal hash碰撞分析
author: ninuxGithub
layout: post
date: 2022-6-20 13:31:32
description: "hash 碰撞分析"
tag: java
---

### 问题分析
    ThreadLocalMap 在set value 的时候有一个for 循环 eg:
    什么时候会执行： e = tab[i = nextIndex(i, len)]

```java
public class ThreadLocal{

    private void set(ThreadLocal<?> key, Object value) {

        Entry[] tab = table;
        int len = tab.length;
        int i = key.threadLocalHashCode & (len-1);

        for (Entry e = tab[i];
             e != null;
             //什么时候会执行这个代码？
             e = tab[i = nextIndex(i, len)]) {
            ThreadLocal<?> k = e.get();

            if (k == key) {
                e.value = value;
                return;
            }
            
            if (k == null) {
                replaceStaleEntry(key, value, i);
                return;
            }
        }

        tab[i] = new Entry(key, value);
        int sz = ++size;
        if (!cleanSomeSlots(i, sz) && sz >= threshold)
            rehash();
    }    
}
```


    为了能让这个执行，需要分析一下代码， 当tab的位置首先不能为null (Entry) 然后要执行第二次才能调用到nextIndex,
    所以要要保证key不同： 随意需要多个ThreadLocal 对象
    要保证k !=null , 那么这个简单
    还需要保证： key.threadLocalHashCode & (len-1) 发生碰撞， 但是从threadLocal 源码来看每次都会增加 （int HASH_INCREMENT = 0x61c88647;）
    那么我们直接通过反射将这个值定为0, 让多次set 发生冲突, 如下代码： 通过修改threadLocalHashCode 来发生hash 碰撞



```java
public class TestDemo{
    /**
     * 探索threadLocal 线性定位，（开放地址探测）
     * @throws Exception
     */
    @Test
    public void testConflict() throws Exception {
        ThreadLocal<Element> threadLocal = new ThreadLocal<>();
        changeField(threadLocal);

        ThreadLocal<Element> threadLocal2 = new ThreadLocal<>();
        changeField(threadLocal2);

        threadLocal.set(new Element().setName("name1").setTag("1").setSeq(1));
        //key 不== 并且  key != null 的时候会调用nextIndex ，线性定位下一个tab的位置
        threadLocal2.set(new Element().setName("name2").setTag("2").setSeq(1));
        threadLocal2.set(new Element().setName("name3").setTag("3").setSeq(1));
    }

    private void changeField(ThreadLocal<Element> threadLocal) throws NoSuchFieldException, IllegalAccessException {
        Field increment = ReflectUtil.getField(ThreadLocal.class, "threadLocalHashCode");
        Field modifiers = Field.class.getDeclaredField("modifiers");
        modifiers.setAccessible(true);
        modifiers.setInt(increment, increment.getModifiers() & ~Modifier.FINAL);
        ReflectUtil.setFieldValue(threadLocal,increment, 0L);
    }
    
}
```

    通过debug 会发现， 第二次for 循环的时候就会调用nextIndex方法了， 会将inedex +1  ， 这就是线下定位（开放地址探测）



### 解决hash冲突的方法
    1.可以采用链表，或者红黑树来保存同一个hash 对应的所有的values  ,  
    2.采用开放地质探测, 上面的例子
    3.采用公共的溢出空间
    
        