---
title: Spring Cloud + @LoadBalanced how to work?
author: ninuxGithub
layout: post
date: 2022-12-20 18:47:48
description: "@LoadBalanced 如何运行的"
tag: spring-cloud
---

## Ribbon 如何实现负载均衡的？ 
    
    代码里面会在RestTemplate bean上面添加了@LoadBalance 就能实现负载均衡， 怎么实现的？
    
    首先spring 会在LoadBalancerAutoConfiguration 里面将我们自己定义的RestTemplate 注入到这个类的内部，那么需要做一下手脚
    主要是通过RestTemplate.addIntercepor() 方法来添加一些自定义的拦截器， 在执行请求的时候， 通过调用这的目标serviceId ,
    Url 等信息， 选择服务器(chooseServer)
    

```java
static class LoadBalancerInterceptorConfig {

		@Bean
		public LoadBalancerInterceptor ribbonInterceptor(
				LoadBalancerClient loadBalancerClient,
				LoadBalancerRequestFactory requestFactory) {
			return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
		}

		@Bean
		@ConditionalOnMissingBean
		public RestTemplateCustomizer restTemplateCustomizer(
				final LoadBalancerInterceptor loadBalancerInterceptor) {
			return restTemplate -> {
				List<ClientHttpRequestInterceptor> list = new ArrayList<>(
						restTemplate.getInterceptors());
				list.add(loadBalancerInterceptor);
                //添加一个spring 负载均衡的拦截器
				restTemplate.setInterceptors(list);
			};
		}

	}
```

    所以其他的动作都是在LoadBalancerInterceptor 内部做了文章 ， 请参考 : https://ninuxgithub.github.io/2018/12/scpost4/

### bean 注入的细节
    当RestTemplate 上面如果没有@LoadBalanced 的注解， 
    spring 通过QualifierAnnotationAutowireCandidateResolver.isAutowireCandidate来判断目标对象， 和spring bean 容器维护的
    beanDifnition 上面是否有对应的 注解， 是否满足， 如果不满足那么注入不了， 所以@LoadBalanced 其实是一个Mark, 来控制是否可以将
    restTemplate 注入到 LoadBalancerAutoConfiguration， 如果能成功注入在对restTemplate就行加工， 就能拥有负载均衡的能力
    否则就是只能直接发送请求
