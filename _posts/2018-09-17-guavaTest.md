---
title: Guava的集合使用
author: ninuxGithub
layout: post
date: 2018-09-17 13:30:06 
description: "java guava 的集合使用方法"
tag: java
---
    
guava个人感觉主要是提供了一下比较优秀的集合框架，例如MultiMap ,  MultiSet , Table, BiMap 等..
直接上代码：
   
```java
package com.example.demo.guava;

import com.google.common.collect.*;

import java.util.Collection;
import java.util.Map;
import java.util.Set;

public class GuavaSet {


    public static void main(String[] args) {
        Multiset<String> multiset = HashMultiset.create();
        multiset.add("a");
        multiset.add("b");
        multiset.add("c");
        multiset.add("d");
        multiset.add("a");
        multiset.add("b");
        multiset.add("c");
        multiset.add("b");
        multiset.add("b");
        multiset.add("b");
        System.out.println("Occurrence of 'b' : " + multiset.count("b"));

        Set<Multiset.Entry<String>> entrySet = multiset.entrySet();
        for (Multiset.Entry<String> entry : entrySet) {
            System.out.println("element : " + entry.getElement() + "(" + entry.getCount() + ")");
        }


        Multimap<String, String> multimap = ArrayListMultimap.create();

        multimap.put("lower", "a");
        multimap.put("lower", "b");
        multimap.put("lower", "c");
        multimap.put("lower", "d");
        multimap.put("lower", "e");

        multimap.put("upper", "A");
        multimap.put("upper", "B");
        multimap.put("upper", "C");
        multimap.put("upper", "D");

        Map<String, Collection<String>> map = multimap.asMap();
        for (String key : map.keySet()) {
            System.out.println(key + ">>");
            Set<Map.Entry<String, Collection<String>>> entries = map.entrySet();
            for (Map.Entry<String, Collection<String>> entry : entries) {
                String k = entry.getKey();
                final Collection<String> value = entry.getValue();
                System.out.println(k + " " + value);
            }

        }

        BiMap<Integer, String> empIDNameMap = HashBiMap.create();

        empIDNameMap.put(new Integer(101), "Mahesh");
        empIDNameMap.put(new Integer(102), "Sohan");
        empIDNameMap.put(new Integer(103), "Ramesh");

        //Emp Id of Employee "Mahesh"
        System.out.println(empIDNameMap.inverse().get("Mahesh"));
        System.out.println(empIDNameMap.get(101));


        //===========================
        Table<String, String, String> employeeTable = HashBasedTable.create();

        //initialize the table with employee details
        employeeTable.put("IBM", "101", "Mahesh");
        employeeTable.put("IBM", "102", "Ramesh");
        employeeTable.put("IBM", "103", "Suresh");

        employeeTable.put("Microsoft", "111", "Sohan");
        employeeTable.put("Microsoft", "112", "Mohan");
        employeeTable.put("Microsoft", "113", "Rohan");

        employeeTable.put("TCS", "121", "Ram");
        employeeTable.put("TCS", "122", "Shyam");
        employeeTable.put("TCS", "123", "Sunil");

        //get Map corresponding to IBM
        Map<String, String> ibmEmployees = employeeTable.row("IBM");

        System.out.println("List of IBM Employees");
        for (Map.Entry<String, String> entry : ibmEmployees.entrySet()) {
            System.out.println("Emp Id: " + entry.getKey() + ", Name: " + entry.getValue());
        }

        //get all the unique keys of the table
        Set<String> employers = employeeTable.rowKeySet();
        System.out.print("Employers: ");
        for (String employer : employers) {
            System.out.print(employer + " ");
        }
        System.out.println();

        //get a Map corresponding to 102
        Map<String, String> EmployerMap = employeeTable.column("102");
        for (Map.Entry<String, String> entry : EmployerMap.entrySet()) {
            System.out.println("Employer: " + entry.getKey() + ", Name: " + entry.getValue());
        }

    }
}



```    
 
    
    
    