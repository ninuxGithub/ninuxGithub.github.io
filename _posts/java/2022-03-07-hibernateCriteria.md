---
title:  Hibernate Criteria Left Join vs subQuery
author: ninuxGithub
layout: post
date: 2022-3-7 17:21:29
description: "hibernate criteria"
tag: java
---



## example


```java
@Entity
@Data
@Table
@ToString
public class Product {

    @Id
    private Long id;

    private String name;

    @OneToMany(mappedBy = "pid", cascade = CascadeType.ALL)
    private List<Attribute> attributes;

}

@Entity
@Data
@Table
@ToString
public class Attribute {

    @Id
    private Long id;

    private Long pid;

    private String info;

    private String date;

}

public class DemoController{

    @RequestMapping(value = "/hibernate")
    public String hibernate() {
        Session session = entityManager.unwrap(SessionImpl.class);
        Criteria criteria = session.createCriteria(Product.class, "pp");
        criteria.setFetchMode("attributes", FetchMode.JOIN);
        criteria.createAlias("attributes", "aat", JoinType.LEFT_OUTER_JOIN);

        criteria.add(Restrictions.ge("aat.date", "2022-01-05"));
        List<Product> list = criteria.list();
        for (Product product : list) {
            System.out.println(product);
        }
        return "ok";
    }

    @RequestMapping(value = "/hibernate2")
    public String hibernate2() {
        Session session = entityManager.unwrap(SessionImpl.class);

        DetachedCriteria criteria = DetachedCriteria.forClass(Product.class);
        //join 会联合成一条sql; 不添加， 那么会形成2条sql
        //criteria.setFetchMode("attributes", FetchMode.JOIN);

        criteria.add(Restrictions.eq("name", "600570"));
        DetachedCriteria dateCriteria = DetachedCriteria.forClass(Attribute.class).add(Restrictions.ge("date", "2022-01-05"));
        DetachedCriteria pidProjection = dateCriteria.setProjection(Projections.property("pid"));
        criteria.add(Subqueries.propertyIn("id", pidProjection));
        Criteria executableCriteria = criteria.getExecutableCriteria(session);
        List<Product> list = executableCriteria.list();
        for (Product product : list) {
            System.out.println(product);
        }
        return "ok";
    }
    
}

```


## Hibernate join vs subQuery
    hibernate 需要建立对象表之间的查询， 重点就是调用createAlias方法， 传递的是一个关联的hibernate 属性字段， 然后取一个别名
    setFetchModd(), createAlias(), 只有hibernate 最清楚对象的关联关系， 不需要考虑类似on a.id = b.id 属性关联的语句， 
    hibernate 内部之间通过Bean 的关联关系进行join
    
    hibernate 的subQuery 其实也很简单， 就是需要用到Projections.property(propertyName) 得到 in (x,y,z) 里面的返回的值
    然后添加一个Subqueries创建in 子查询就好了， 记录以备查询

    
    
