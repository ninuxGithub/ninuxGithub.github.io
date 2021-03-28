---
title: Spring Bean 循环依赖
author: ninuxGithub
layout: post
date: 2021-3-28 14:02:01
description: "spring bean循环依赖的探索"
tag: spring
---

##### spring bean循环依赖的探索

###### 创建类进行测试

在spring 里面通常会遇到bean 在创建的时候需要了循环依赖的情况， 例如

a 类里面依赖的b

b类里面依赖的a

这样循环依赖的场景

首先我们来创建2个非常简单的类用来后续的debug探索.

```java
@Component
public class A {

    @Autowired
    private B b;
}


@Component
public class B {

    @Autowired
    private A a;
}

```



###### debug 分析解决循环的方法

然后就是合适的下端的地方和方式， 可以帮助我们清楚的了解过程

org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean

```java
//加入断点 ，condition : "a".equalsIgnoreCase(beanName)||"b".equalsIgnoreCase(beanName)
Object sharedInstance = getSingleton(beanName);

//加入断点 ，condition : "a".equalsIgnoreCase(beanName)||"b".equalsIgnoreCase(beanName)
sharedInstance = getSingleton(beanName, () -> {
    try {
    return createBean(beanName, mbd, args);
    }
    catch (BeansException ex) {
    // Explicitly remove instance from singleton cache: It might have been put there
    // eagerly by the creation process, to allow for circular reference resolution.
    // Also remove any beans that received a temporary reference to the bean.
    destroySingleton(beanName);
    throw ex;
    }
});
```



org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean

```java
boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
      isSingletonCurrentlyInCreation(beanName));
//判断是否提前暴露对象， 加入到singletonFactory 里面去，非常重要
if (earlySingletonExposure) {
   if (logger.isDebugEnabled()) {
      logger.debug("Eagerly caching bean '" + beanName +
            "' to allow for resolving potential circular references");
   }
   addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
}

// Initialize the bean instance.
Object exposedObject = bean;
try {
//加入断点 ，condition : "a".equalsIgnoreCase(beanName)||"b".equalsIgnoreCase(beanName)    
   populateBean(beanName, mbd, instanceWrapper);
   exposedObject = initializeBean(beanName, exposedObject, mbd);
}
catch (Throwable ex) {
   //...
}
```

org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor.AutowiredFieldElement#inject

注入b feild的时候去spring 容器寻找是否有对应的bean

```java
try {
   value = beanFactory.resolveDependency(desc, beanName, autowiredBeanNames, typeConverter);
}
//...
instanceCandidate = descriptor.resolveCandidate(autowiredBeanName, type, this);
//获取b , 发现没有就去创建b
beanFactory.getBean(beanName);
```



###### 分析bean如何完成的创建和注入

1. 创建a, 实例化bean , 然后提前暴露自己保存到singletonFactories 里面去， 为后去依赖a的bean的创建做好准备， 这是解决循环依赖的一个重要的点

2. populateBean 方法执行， 开始填充A 里面的属性b, getBean b 发现没有， 那么就创建b, 同样的道理还是实例化B ， 然后提前暴露放到三级缓存里面

3. populateBean 方法执行， 开始填充B里面的属性a, getBean a 发现有了， 上面已经提前暴露了a 到三级缓存（singletonFactories）然后三级缓存移除a, 并且放入到二级缓存（earlySingletonObjects  保存的就是earlyBeanReference）

4. 然后注入a feild 到 B 的属性里面去， 完成后在将b从三级缓存移除， 放入到了一级缓存，然后实例化b的操作

5. 然后注入b feild 到A里面去， 直接从一级缓存就可以获取到了b ，然后将二级缓存的a移除放入到一级缓存里面去， 注入b 后 A就完整了， 然后进行实例化的操作

6. ![image-bean-circle-inject](/images/posts/bean-circle-inject.png)

   
    
    
    
    
 
    
    
    