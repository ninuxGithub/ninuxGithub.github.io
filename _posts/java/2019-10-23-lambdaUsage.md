---
title: java 1.8 lambda 用法心得
author: ninuxGithub
layout: post
date: 2019-10-23 08:59:59
description: "lambda"
tag: java
---

## lambda usage

    Predicate 在对集合的筛选的时候是非常有用的
    
    ToDoubleFunction 可以动态的获取我们的javabean 里面的属性的值


```java
package com.example.study.lambda;

import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import java.util.function.*;

/**
 * @author shenzm
 * @ate 2019-10-22 8:35
 * @since 1.0
 */
public class LambdaTest {

    public static void main(String[] args) {
        List<DailyQuote> list = new ArrayList<>();
        list.add(new DailyQuote(12.52, 13.00, 12.50, 11.44, "京东方A"));
        list.add(new DailyQuote(22.45, 23.52, 18.22, 29.56, "京东方B"));
        list.add(new DailyQuote(25.52, 89.78, 23.22, 14.44, "京东方C"));
        list.add(new DailyQuote(22.22, 22.22, 22.22, 22.22, null));


        //predicate使用
        predicate(list);

        //ToDoubleFunction 获取特定的字段
        getField(list);


        //LongFunction
        LongFunction<Long> lf = Long::valueOf;
        Long apply = lf.apply(12L);
        System.out.println(apply);


        //有入参，无结果
        Consumer<String> consumer = System.out::println;
        consumer.accept("consumer java");


        //无入参， 有结果
        Supplier<String> supplier = () -> "supplier java";
        System.out.println(supplier.get());

        //有入参， 有结果
        Function<String, String> function = s -> "function " + s;
        System.out.println(function.apply("函数测试"));

        //有入参， 出参为boolean
        Predicate<Integer> p = x -> x > 0;
        System.out.println(p.test(2));


        testOptional(list);

    }

    /**
     * 测试optional
     *
     * @param list 数据
     */
    public static void testOptional(List<DailyQuote> list) {
        //optional + orElse(Object elseValue)
        for (DailyQuote quote : list) {
            String s = Optional.ofNullable(quote)
                    .map(DailyQuote::getSecuAbbr)
                    .map(String::toUpperCase)
                    .orElse(null);
            System.out.println(s);
        }

        //optional + orElseGet(Supplier supplier)
        for (DailyQuote quote : list) {
            String s = Optional.ofNullable(quote)
                    .map(DailyQuote::getSecuAbbr)
                    .map(String::toUpperCase)
                    .orElseGet(new Supplier<String>() {
                        @Override
                        public String get() {
                            return "defaultValue";
                        }
                    });
            System.out.println(s);
        }


        //optional + orElseThrow(Supplier exceptionSupplier)
        for (DailyQuote quote : list) {
            String s = null;
            try {
                s = Optional.ofNullable(quote)
                        .map(DailyQuote::getSecuAbbr)
                        .map(String::toUpperCase)
                        .orElseThrow(new Supplier<Throwable>() {
                            @Override
                            public Throwable get() {
                                return new RuntimeException("空指针");
                            }
                        });
            } catch (Throwable throwable) {
                //throwable.printStackTrace();
            }
            System.out.println(s);
        }
    }

    /**
     * 通过ToDoubleFunction来动态的获取javabean中的某个字段， 在实际的开发中有重要用处
     *
     * @param list 数据
     */
    private static void getField(List<DailyQuote> list) {
        ToDoubleFunction<DailyQuote> closePriceFunction = DailyQuote::getClosePrice;
        ToDoubleFunction<DailyQuote> openPriceFunction = DailyQuote::getOpenPrice;

        for (DailyQuote quote : list) {
            double closePrice = closePriceFunction.applyAsDouble(quote);
            double openPrice = openPriceFunction.applyAsDouble(quote);
            System.out.println(closePrice + " " + openPrice);
        }
    }

    /**
     * 通过predicate来获取closePrice > 10 且 highPrice < 15 的记录
     *
     * @param list 数据
     */
    private static void predicate(List<DailyQuote> list) {
        Predicate<DailyQuote> predicate = new Predicate<DailyQuote>() {
            @Override
            public boolean test(DailyQuote dailyQuote) {
                if (null != dailyQuote) {
                    if (dailyQuote.getClosePrice() > 10 && dailyQuote.getHighPrice() < 15) {
                        return true;
                    }
                }
                return false;
            }
        };

        //Predicate 使用： 相当于动态的过滤的条件来获取记录
        for (DailyQuote quote : list) {
            if (predicate.test(quote)) {
                System.out.println(quote);
            }
        }
    }
}


```