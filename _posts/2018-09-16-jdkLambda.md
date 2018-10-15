---
title: jdk1.8 lambda 的使用
author: ninuxGithub
layout: post
date: 2018-09-16 17:30:06 
description: "java lambda的各种使用demo"
tag: java
---
    
   
直接上代码：
   
```java

package com.example.demo.lambda;

import java.util.*;
import java.util.function.Function;
import java.util.stream.Collectors;

public class TestLambda {

    public static void main(String[] args) {
        List<Student> list = new ArrayList<>();
        list.add(new Student("tom", 18, "男", 1500, 1.6));
        list.add(new Student("ben", 18, "男", 1200, 1.3));
        list.add(new Student("lily", 18, "女", 1100, 1.2));
        list.add(new Student("啊雅", 18, "女", 1400, 1.5));
        list.add(new Student("啊雅2", 18, "女", 1600, 1.5));
        list.add(new Student("王丽", 18, "女", 1800, 1.1));
        list.add(null); //--------->null 元素


        Map<String, Student> map = new HashMap<>();
        map.put("001", new Student("码农", 18, "男", 2000, 1.3));
        map.put("002", new Student("小芳", 18, "女", 3000, 1.5));
        Map<Integer, Student> sortMap = new HashMap<>();
        sortMap.put(600570, new Student("码农", 18, "男", 2000, 1.3));
        sortMap.put(400570, new Student("小芳", 18, "女", 3000, 1.5));
        sortMap.put(500570, new Student("小芳", 18, "女", 3000, 1.5));


        //reduce求和
        Double sumUp = list.stream().filter(Objects::nonNull).map(v -> v.getSalary()).reduce((x, y) -> x + y).get();
        System.out.println("reduce sum : " + sumUp);


        //获取排序后的map 的key
        List<Integer> sortedKey = sortMap.keySet().stream().sorted((Integer n1, Integer n2) -> n1 > n2 ? 1 : -1).collect(Collectors.toList());
        System.out.println("sortedKey " + sortedKey);

        //joining 的使用
        String keyJoin = sortMap.keySet().stream().sorted((Integer n1, Integer n2) -> n1 > n2 ? 1 : -1).map(v -> v.toString()).collect(Collectors.joining(","));
        System.out.println("keyJoin " + keyJoin);


        //获取排序后的整个key : value 的  entry对象
        List<Map.Entry<Integer, Student>> sortedList = sortMap.entrySet().stream()
                .sorted((Map.Entry<Integer, Student> n1, Map.Entry<Integer, Student> n2) -> n1.getKey() > n2.getKey() ? 1 : -1)
                .collect(Collectors.toList());
        //sortedList.stream().forEach(System.out::println);
        sortedList.stream().forEach(v -> System.out.println("根据map的key正序排序： " + v));

        //测试空集合
        List<Student> nullList = new ArrayList<>();
        nullList.add(null);

        //排除非空
        Optional.ofNullable(nullList).orElse(Collections.emptyList()).stream().filter(Objects::nonNull).forEach(v -> System.out.println("vvvvvvv" + v));


        //获取学生的姓名的名单
        getStudentNames(list);


        //获取富有的学生
        getWealthyList(list);

        //获取女孩的集合
        girlLIst(list);

        //转变为list
        convert2List(map);


        //获取工作属性的最大工资
        Double maxSalary = list.stream().filter(Objects::nonNull).map(v -> v.getSalary()).max((Double d1, Double d2) -> d1 > d2 ? 1 : -1).get();
        System.out.println("maxSalary : +" + maxSalary);


        Student maxSalStudent = list.parallelStream().filter(Objects::nonNull).max((Student s1, Student s2) -> s1.getSalary() > s2.getSalary() ? 1 : -1).get();
        System.out.println("maxSalStudent" + maxSalStudent);

        // list 转map
        Map<String, Student> name2StuMap = list.stream().filter(Objects::nonNull).collect(Collectors.toMap(Student::getName, Function.identity(), (k1, k2) -> k1));


        //filter(Objects::nonNull) 过滤空的元素
        name2StuMap.entrySet().stream().filter(Objects::nonNull).forEach(entry -> System.out.println(entry));


    }

    private static void convert2List(Map<String, Student> map) {
        List<Student> convert2List = map.entrySet().stream().map(entry -> {
            Student s = entry.getValue();
            s.setName(entry.getKey());
            return s;
        }).collect(Collectors.toList());
        System.out.println("list" + convert2List);
    }

    private static void girlLIst(List<Student> list) {
        List<String> girlsList = list.stream().filter(Objects::nonNull).filter(f -> f.getSex().equals("女")).map(m -> m.getName()).collect(Collectors.toList());
        System.out.println("girlList" + girlsList);
    }

    private static void getWealthyList(List<Student> list) {
        List<Wealthy> wealthyList = list.stream().filter(Objects::nonNull).filter(v -> v.getSalary() > 1500).map(v -> {
            Wealthy wealthy = new Wealthy();
            wealthy.setSalary(v.getSalary());
            wealthy.setName(v.getName());
            wealthy.setAge(v.getAge());
            return wealthy;
        }).collect(Collectors.toList());
        System.out.println("富有的学生：" + wealthyList);
    }

    private static void getStudentNames(List<Student> list) {
        //map 操作
        List<String> nameList = list.stream().filter(Objects::nonNull).map(v -> {
            return v.getName();
        }).collect(Collectors.toList());

        //排序操作
        nameList.sort((String n1, String n2) -> ChinsesUtil.convertHanzi2Int(n1) < ChinsesUtil.convertHanzi2Int(n2) ? -1 : 1);
        System.out.println("学生的姓名名单：" + nameList);
    }
}



```    
 
    
    
    