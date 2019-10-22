---
title: mybatis对象多对一双向绑定
author: ninuxGithub
layout: post
date: 2019-10-22 13:48:15
description: "mybatis关联"
tag: mybatis
---


## mybatis 如何实现对象的相互关联
    
    多端采用collection来描述“一端”；
    
    一端采用association来描述“多端”
    
    建立javaBean如下
  
  
```java
/**
 * @author shenzm
 * @date 2019-10-22 11:08
 * @since 1.0
 */
public class Depart {

    private Long did;

    private String departName;

    private List<Employee> employees;

    public Long getDid() {
        return did;
    }

    public void setDid(Long did) {
        this.did = did;
    }

    public String getDepartName() {
        return departName;
    }

    public void setDepartName(String departName) {
        this.departName = departName;
    }

    public List<Employee> getEmployees() {
        return employees;
    }

    public void setEmployees(List<Employee> employees) {
        this.employees = employees;
    }

    @Override
    public String toString() {
        return "Depart{" +
                "did=" + did +
                ", departName='" + departName + '\'' +
                ", employees=" + employees +
                '}';
    }
}



/**
 * @author shenzm
 * @date 2019-10-22 11:09
 * @since 1.0
 */
public class Employee {

    private Long eid;

    private String name;

    private int age;

    private Depart depart;

    public Long getEid() {
        return eid;
    }

    public void setEid(Long eid) {
        this.eid = eid;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public Depart getDepart() {
        return depart;
    }

    public void setDepart(Depart depart) {
        this.depart = depart;
    }

    @Override
    public String toString() {
        return "Employee{" +
                "eid=" + eid +
                ", name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
    
    /*
    
    如果需要重写toString方法， 不可相互定义， 不然会造成栈调用递归 stackOverFlowError 
    @Override
        public String toString() {
            return "Employee{" +
                    "eid=" + eid +
                    ", name='" + name + '\'' +
                    ", age=" + age +
                    ", depart=" + depart +
                    '}';
        }
        
        
     */
}

```    

        mapper的xml配置

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.study.mapper.DepartMapper">

    <resultMap type="com.example.study.bean.Depart" id="departResultMap">
        <id column="did" property="did" javaType="java.lang.Long"/>
        <result column="departName" property="departName" javaType="java.lang.String"/>
        <!--通过collection来映射list类型的属性， 通过select来查询关联的对象-->
        <collection property="employees"
                    ofType="com.example.study.bean.Employee"
                    select="com.example.study.mapper.EmployeeMapper.getEmployees"
                    column="did"/>
    </resultMap>

    <select id="findAll" resultType="java.util.List" resultMap="departResultMap">
        <![CDATA[
        select * from depart
        ]]>
    </select>

    <select id="selectRowById" resultType="com.example.study.bean.Depart" resultMap="departResultMap">
        <![CDATA[
        select * from depart where did=#{did}
        ]]>
    </select>


</mapper>



<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.study.mapper.EmployeeMapper">

    <resultMap type="com.example.study.bean.Employee" id="employeeResultMap">
        <id column="eid" property="eid" javaType="java.lang.Long"/>
        <result column="name" property="name" javaType="java.lang.String"/>
        <result column="age" property="age" javaType="int"/>

        <!-- 通过association来进行一对多， 来描述“多”的对象的属性， 通过select来查询对象-->
        <association property="depart" javaType="com.example.study.bean.Depart"
                     select="com.example.study.mapper.DepartMapper.selectRowById" column="did"/>
        <!--
                <association property="depart" javaType="com.example.study.bean.Depart"
                             select="com.example.study.mapper.DepartMapper.selectRowById" column="did">
                    没有必选定义Depart类的信息
                    <id column="did" property="did" javaType="java.lang.Long"/>
                    <result column="departName" property="departName" javaType="java.lang.String"/>
                </association>
        -->
    </resultMap>

    <select id="getEmployees" resultType="java.util.List" resultMap="employeeResultMap">
        <![CDATA[

                      select * from employee where did = #{did}

        ]]>
    </select>


</mapper>
```


    需要准备数据
    
```sql
/*
Navicat MySQL Data Transfer

Source Server         : mysql
Source Server Version : 50556
Source Host           : 10.1.51.96:3306
Source Database       : test

Target Server Type    : MYSQL
Target Server Version : 50556
File Encoding         : 65001

Date: 2019-10-22 13:44:36
*/

SET FOREIGN_KEY_CHECKS=0;

-- ----------------------------
-- Table structure for depart
-- ----------------------------
DROP TABLE IF EXISTS `depart`;
CREATE TABLE `depart` (
  `did` bigint(11) NOT NULL,
  `departName` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`did`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

-- ----------------------------
-- Records of depart
-- ----------------------------
INSERT INTO `depart` VALUES ('1', '研发部');
INSERT INTO `depart` VALUES ('2', '销售部');

-- ----------------------------
-- Table structure for employee
-- ----------------------------
DROP TABLE IF EXISTS `employee`;
CREATE TABLE `employee` (
  `eid` bigint(20) NOT NULL,
  `name` varchar(255) DEFAULT NULL,
  `age` int(11) DEFAULT NULL,
  `did` bigint(20) DEFAULT NULL,
  PRIMARY KEY (`eid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

-- ----------------------------
-- Records of employee
-- ----------------------------
INSERT INTO `employee` VALUES ('1', '张三', '11', '1');
INSERT INTO `employee` VALUES ('2', '李四', '12', '1');
INSERT INTO `employee` VALUES ('3', '王五', '13', '2');
INSERT INTO `employee` VALUES ('4', '赵柳', '11', '3');

```    



##重点
    在一端的表里面需要持续多的一端的id； toString方法避免栈溢出，循环打印对象；
    