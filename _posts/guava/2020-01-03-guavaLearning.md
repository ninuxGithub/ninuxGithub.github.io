---
title: guava学习
author: ninuxGithub
layout: post
date: 2020-1-3 09:22:31
description: "guava集合"
tag: guava
---


### guava介绍
    guava是google推出的一个java集合框架，对java.util.collection包进行了再次的封装， 提供了更加便捷的结合工具
    例如Lists,Maps,multiset, multimap, bimap,table等等
    
    
    multiset可以理解为在set的基础上对相同的元素进行了统计（count）, 有elementSet方法列出不同的key的set集合
    
    multilist就是Map<Object>,List<Object>> 相同的key的元素会被放到一个List中去， 不需要些复杂的泛型机构
    
    bimap 简单的理解为可以通过调用inverse方法后， 通过value来索引key （倒转了索引的顺序）‘’‘
    
    table (x, y)-->value ; 简单的理解就是一个坐标系， 根据x轴和y轴可以定位到某个元素；
 
```java
package com.derbysoft.guava;

import com.google.common.collect.*;

import java.util.Collection;
import java.util.Map;

/**
 * @author thunderstorm.shen on 2020/1/2
 */
public class CollectionTest {

    public static void main(String[] args) {
        multiset();

        multimap();

        bimap();

        table();


    }

    /**
     * <table width="600px" border="1px"  cellspacing="0" cellpadding="0" style="text-align: center; border-color: red">
     * <tr>
     *     <td></td>
     *     <td>A</td>
     *     <td>B</td>
     *     <td>C</td>
     * </tr>
     * <tr>
     *     <td>001</td>
     *     <td>java</td>
     *     <td>python</td>
     *     <td></td>
     * </tr>
     * <tr>
     *     <td>002</td>
     *     <td>c++</td>
     *     <td></td>
     *     <td></td>
     * </tr>
     * <tr>
     *     <td>003</td>
     *     <td></td>
     *     <td>groovy</td>
     *     <td></td>
     * </tr>
     * <tr>
     *     <td>004</td>
     *     <td></td>
     *     <td></td>
     *     <td>android</td>
     * </tr>
     *
     * </table>
     */
    private static void table() {
        Table<String, String, String> table = HashBasedTable.create();
        table.put("001", "A", "java");
        table.put("001", "B", "python");
        table.put("002", "A", "c++");
        table.put("003", "B", "groovy");
        table.put("004", "C", "android");

        Map<String, String> row = table.row("001");
        System.out.println("001 row : " + row);

        Map<String, String> column = table.column("A");
        System.out.println("A column: " + column);


        Collection<String> values = table.values();
        System.out.println("values : " + values);

        System.out.println(table);
    }

    private static void bimap() {
        BiMap<String, String> biMap = HashBiMap.create();
        biMap.put("0001", "tom");
        biMap.put("0002", "jack");
        biMap.put("0003", "bob");
        //bimap inverse 之后可以从value来索引key
        System.out.println(biMap.inverse().get("tom"));
    }

    private static void multimap() {
        Multimap<String, String> multimap = ArrayListMultimap.create();
        multimap.put("lower", "a");
        multimap.put("lower", "b");
        multimap.put("lower", "c");
        multimap.put("lower", "d");

        multimap.put("upper", "1");
        multimap.put("upper", "2");
        multimap.put("upper", "3");
        multimap.put("upper", "4");

        System.out.println(multimap);

        System.out.println(multimap.get("lower"));
        System.out.println(multimap.get("upper"));

        Map<String, Collection<String>> map = multimap.asMap();
        for (String key : map.keySet()) {
            System.out.println(key + " " + map.get(key));

        }
    }

    private static void multiset() {
        Multiset<String> multiset = HashMultiset.create();
        multiset.add("a");
        multiset.add("a");
        multiset.add("b");
        multiset.add("b");
        multiset.add("c");
        multiset.add("c");
        multiset.add("d");

        System.out.println(multiset.size());
        System.out.println(multiset.elementSet());
        System.out.println(multiset);
    }
}

```    


    还有一个是对future进行了封装， 在futrue的基础上添加了监听器， callback成功和失败的回调函数；
    
    for example: addListener方法对ListenableFutrue添加了一个Runnable任务进行监听；
                 addCallback方法 是通过Futures静态方法来为ListenableFuture添加了一个FutrueCallback
    
    
```java
package com.derbysoft.guava;

import com.derbysoft.common.util.ThreadUtils;
import com.google.common.collect.Lists;
import com.google.common.util.concurrent.*;
import org.checkerframework.checker.nullness.qual.Nullable;
import org.springframework.util.StopWatch;

import java.util.List;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * refer github : https://github.com/google/guava/wiki/ListenableFutureExplained
 * guava guide: https://www.tutorialspoint.com/guava/guava_overview.htm
 *
 * @author thunderstorm.shen
 *         Created by thunderstorm.shen on 2019/12/31.
 */
public class ThreadPoolListener {

    public static void main(String[] args) {
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(5, 8, 100,
                TimeUnit.MILLISECONDS,
                new ArrayBlockingQueue<>(1000), new ThreadPoolExecutor.CallerRunsPolicy());
        ListeningExecutorService listeningExecutorService = MoreExecutors.listeningDecorator(threadPoolExecutor);
        List<String> list = Lists.newArrayList("java", "c++", "php");

        addListener(threadPoolExecutor, listeningExecutorService, list);
        addCallback(listeningExecutorService, list);

        ThreadUtils.sleep(5, TimeUnit.SECONDS);
        listeningExecutorService.shutdownNow();


    }

    private static void addCallback(ListeningExecutorService listeningExecutorService, List<String> list) {
        for (String s : list) {
            ListenableFuture<String> future = listeningExecutorService.submit(new Callable<String>() {
                @Override
                public String call() throws Exception {
                    ThreadUtils.sleep(2, TimeUnit.SECONDS);
                    return s.toUpperCase();
                }
            });

            Futures.addCallback(future, new FutureCallback<String>() {
                @Override
                public void onSuccess(@Nullable String result) {
                    System.out.println("onSuccess : " + result + " " + Thread.currentThread().getName());
                }

                @Override
                public void onFailure(Throwable t) {
                    System.out.println("task is failed ");
                }
            }, listeningExecutorService);
        }
    }

    private static void addListener(ThreadPoolExecutor threadPoolExecutor, ListeningExecutorService listeningExecutorService, List<String> list) {
        final AtomicInteger num = new AtomicInteger(list.size());
        StopWatch stopWatch = new StopWatch("guava-listener");
        stopWatch.start();
        for (String s : list) {
            ListenableFuture<String> future = listeningExecutorService.submit(new Callable<String>() {
                @Override
                public String call() throws Exception {
                    ThreadUtils.sleep(2, TimeUnit.SECONDS);
                    return s.toUpperCase();
                }
            });

            future.addListener(new Runnable() {
                @Override
                public void run() {
                    try {
                        num.getAndDecrement();
                        System.out.println(future.get());
                        System.out.println(num.get());
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } catch (ExecutionException e) {
                        e.printStackTrace();
                    }
                }
            }, listeningExecutorService);
        }

        if (num.get() == 0) {
            threadPoolExecutor.shutdown();
            stopWatch.stop();
            System.out.println(stopWatch.prettyPrint());
        }
    }

}

```    

 
    guava还提供了一种cache的方法, 可以设置expire时间， cache的capacity容量可以从DB加载数据缓存到内存里面去；    

```java
package com.derbysoft.guava;

import com.google.common.cache.CacheBuilder;
import com.google.common.cache.CacheLoader;
import com.google.common.cache.LoadingCache;

import java.util.UUID;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.TimeUnit;

/**
 * @author thunderstorm.shen on 2020/1/2
 */
public class CacheTest {
    public static void main(String[] args) throws ExecutionException {
        LoadingCache<String, MemoryCache> cache = CacheBuilder.newBuilder().maximumSize(100)
                .expireAfterAccess(30, TimeUnit.MINUTES)
                .build(new CacheLoader<String, MemoryCache>() {
                    @Override
                    public MemoryCache load(String key) throws Exception {
                        return new MemoryCache(key, UUID.randomUUID().toString());
                    }
                });

        System.out.println(cache.get("java"));
        cache.refresh("java");
    }


    static class MemoryCache {

        private String key;

        private Object value;

        public MemoryCache(String key, Object value) {
            this.key = key;
            this.value = value;
        }

        public String getKey() {
            return key;
        }

        public void setKey(String key) {
            this.key = key;
        }

        public Object getValue() {
            return value;
        }

        public void setValue(Object value) {
            this.value = value;
        }

        @Override
        public String toString() {
            final StringBuffer sb = new StringBuffer("MemoryCache{");
            sb.append("key='").append(key).append('\'');
            sb.append(", value=").append(value);
            sb.append('}');
            return sb.toString();
        }
    }
}

```
    
    
    