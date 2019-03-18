---
title: spring boot + dubbo
author: ninuxGithub
layout: post
date: 2019-3-18 11:24:06
description: "spring boot + dubbo"
tag: dubbo
---


### 目的
    采用spring boot 整合dubbo, 配置采用yaml， bean 采用java config 
    废话不多说，直接上代码 重要的地方会有注释
    
    
    项目寄托在github : https://github.com/ninuxGithub/spring-boot-dubbo-zookeeper.git
    
    
    实现的思路和dubbo xml config 差不多 都是采用api提供接口， provider实现接口 ， consumer调用接口
    
    
    
    在provider 里面 需要对api接口的实现，实现的demo如下
    
    采用注解：
        @Service(
                version = "${spring.application.version}",
                application = "${dubbo.application.id}",
                protocol = "${dubbo.protocol.id}",
                registry = "${dubbo.registry.id}"
        )
    version , application, protocol, registry 都是通过yaml 配置的
    

```java
import com.alibaba.dubbo.config.annotation.Service;
import com.example.api.service.HelloService;

/**
 * @author shenzm
 * @date 2019-2-11
 * @description 作用
 */

@Service(
        version = "${spring.application.version}",
        application = "${dubbo.application.id}",
        protocol = "${dubbo.protocol.id}",
        registry = "${dubbo.registry.id}"
)
public class DefaultHellowService implements HelloService {
    @Override
    public String sayHello(String name) {
        return "hello " + name + "  welcome!";
    }

    @Override
    public String sayGoodbye(String name) {
        return "byebye " + name + " ,see u !";
    }
}
```


    在consumer 对接口进行调用
    调用：
        
        @Reference(
                version = "${spring.application.version}",
                application = "${dubbo.application.id}",
                registry = "${dubbo.registry.id}"
        )
     
    获取到dubbo的service服务提供对象
    
        
    
```java
@RestController
public class HelloController {


    @Reference(
            version = "${spring.application.version}",
            application = "${dubbo.application.id}",
            registry = "${dubbo.registry.id}"
    )
    private HelloService helloService;

    @RequestMapping(value ="/hello/{name}")
    public String sayHello(@PathVariable String name){
        return helloService.sayHello(name);
    }


    @RequestMapping(value="/sayGoodbye/{name}")
    public String sayGoodbye(@PathVariable String name){
        return helloService.sayGoodbye(name);
    }
}
```



### 遇到过的坑
    在provider 部分 yaml的配置必须如下 不然会扫描不到package, 扫不到那些bean加入了@Service注解 无法注册服务

```yaml
dubbo:
  scan:
    basePackages: com.example.provider.service
```