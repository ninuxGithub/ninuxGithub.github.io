---
title: SpringCloud Gateway使用
author: ninuxGithub
layout: post
date: 2019-8-6 09:27:06
description: "SpringCloud Gateway"
tag: spring-cloud
---

## spring cloud gateway 
    gateway的作用是可以来作为服务的一个代理， 请求的转发，中间的一个路由的功能， 这就好比是nginx的功能；
    相比较zuul, zuul是将请求结果代理转发到了当前eureka注册中心的其他的服务上去的， 局限在了本服务集群体系里面的服务调用了；
    
    
    gateway有跟好的一个路由的功能；
    
    本章的目的是为了记录gateway一些简单的使用， 内容参考了其他的博客， 自己实践一下；
    
    
```java
@SpringBootApplication
@RestController
@EnableDiscoveryClient
public class GatewaySimpleApplication {

    public static void main(String[] args) {
        SpringApplication.run(GatewaySimpleApplication.class, args);
    }

    @Autowired
    ConfigurableApplicationContext applicationContext;

    @Bean
    public RouteLocatorBuilder routeLocatorBuilder() {
        return new RouteLocatorBuilder(applicationContext);
    }

    //全局的token的过滤器
    @Bean
    public TokenFilter tokenFilter() {
        return new TokenFilter();
    }

    //自定义的resolver来给redis-rate-limiter使用
    @Bean
    public AddrKeyResolver addrKeyResolver(){
        return new AddrKeyResolver();
    }

    @Bean
    public UriKeyResolver uriKeyResolver(){
        return new UriKeyResolver();
    }


    @Bean
    public RouteLocator routeLocator(RouteLocatorBuilder builder) {
        String httpUrl = "http://httpbin.org:80/get";
        return builder.routes()

                //路由的参数restfull请求
                .route(r -> r.path("/foo/{segment}").uri(httpUrl).id("route_segment"))

                //cookie参数
                .route(r -> r.cookie("name", "java").uri(httpUrl).id("cookie_test"))

                //header参数
                .route(p -> p.path("/get").filters(f -> f.addRequestHeader("Hello", "World")).uri(httpUrl).id("header_param"))

                //时间before
                .route(r -> r.before(changeShanghaiToUTC("2020-01-01 11:11:11")).uri(httpUrl).id("before_time"))

                //时间after
                .route(r -> r.after(changeShanghaiToUTC("2018-01-01 11:11:11")).uri(httpUrl).id("after_time"))

                //fallback
                .route(p -> p.host("*.hystrix.com").filters(f -> f.hystrix(h -> h.setName("cmd").setFallbackUri("forward:/fallback"))).uri(httpUrl).id("fallback"))

                //order越大优先级越低
                .route(r -> r.path("/customer/**")
                        .filters(f -> f.filter(new RequestTimeFilter()).addRequestHeader("X-Response-Default-Foo", "Default-Foo"))
                        .uri(httpUrl).order(-1).id("customer_filter"))

                .build();
    }

    /**
     * 请求的时间计算的过滤器
     */
    static class RequestTimeFilter implements GatewayFilter, Order {
        private static final Logger logger = LoggerFactory.getLogger(RequestTimeFilter.class);

        static String TEMP_KEY = "startTime";

        @Override
        public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
            exchange.getAttributes().put(TEMP_KEY, System.currentTimeMillis());
            return chain.filter(exchange).then(Mono.fromRunnable(() -> {
                Long startTime = exchange.getAttribute(TEMP_KEY);
                if (null != startTime) {
                    logger.info(exchange.getRequest().getURI().getRawPath() + " : " + (System.currentTimeMillis() - startTime));
                }
            }));
        }

        @Override
        public int value() {
            return 0;
        }

        @Override
        public Class<? extends Annotation> annotationType() {
            return null;
        }
    }

    /**
     * token 全局的过滤器
     */
    static class TokenFilter implements GlobalFilter, Order {

        private static final Logger logger = LoggerFactory.getLogger(TokenFilter.class);

        @Override
        public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
            String token = exchange.getRequest().getQueryParams().getFirst("token");
            if (StringUtils.isEmpty(token)) {
                logger.info("token is empty...");
                exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                return exchange.getResponse().setComplete();
            }
            return chain.filter(exchange);
        }

        @Override
        public int value() {
            return 0;
        }

        @Override
        public Class<? extends Annotation> annotationType() {
            return null;
        }
    }

    static class AddrKeyResolver implements KeyResolver{

        @Override
        public Mono<String> resolve(ServerWebExchange exchange) {
            return Mono.just(exchange.getRequest().getRemoteAddress().getAddress().getHostAddress());
        }
    }

    static class UriKeyResolver implements KeyResolver{
        @Override
        public Mono<String> resolve(ServerWebExchange exchange) {
            return Mono.just(exchange.getRequest().getURI().getPath());
        }
    }


    public static ZonedDateTime changeShanghaiToUTC(String dateStr) {
        DateTimeFormatter beijingFormatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss").withZone(ZoneId.of("Asia/Shanghai"));
        if (StringUtils.isBlank(dateStr)) {
            return null;
        }
        ZonedDateTime beijingDateTime = ZonedDateTime.parse(dateStr, beijingFormatter);
        return beijingDateTime.withZoneSameInstant(ZoneId.of("UTC"));
    }


    /**全局的fallback*/
    @RequestMapping("/fallback")
    public Mono<String> fallback() {
        System.out.println("fallback run...");
        return Mono.just("fallback");
    }


}

```    



    另外还有一种route是通过yaml来进行配置的， 无论是java config 还是 yaml配置都是可以的
    描述：     
        limite_route，当请求http://localhost:8081/getMessages?messageID=2&token=222服务的时候会进行限流的动作
        这就是redis-rate-limiter的使用， 之前一直是不生效，经过搜索无法找到正解， 最后查看日志发现走了其他的route, 
        route都是可以自定义id的， 我们可以区分当前的请求是结果了那个route , 不生效是因为这个route的优先级别不够被别的route限制性了；
        解决办法：加入order =-1 或者进行路由path进行匹配精准一些；
        
        hystrix_route是个请求hystrxy调用失败的测试
        
        启用gateway的endpoint
    
    
    
    
```yaml

server:
  port: 8081

logging:
  level:
    org.springframework.cloud.gateway: trace

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
    should-enforce-registration-at-init: false
    register-with-eureka: true
    fetch-registry: true
    on-demand-update-status-change: true

spring:
  redis:
    host: 10.1.51.96
    password: redis
    pool: 6379
    database: 0
  cloud:
    gateway:
      routes:
      #http://localhost:8081/getMessages?messageID=2&token=222
      - id: limite_route
        uri: lb://EURKA-CLIENT
        order: -1
        predicates:
        - Path=/getMessages/**
        filters:
        - name: RequestRateLimiter
          args:
            redis-rate-limiter:
              replenishRate: 1  # 用户每秒完成请求的个数
              burstCapacity: 1  #1s完成的最大的请求个数
            key-resolver: "#{@addrKeyResolver}"

      #http://localhost:8081/hi?name=java&token=222
      - id: hystrix_route
        uri: lb://EURKA-CLIENT
        order: -1
        predicates:
        - Path=/hi
        filters:
        - RewritePath=/hi/(?<segment>.*),/$\{segment}
        - name: Hystrix
          args:
            name: fallbackcmd
            fallbackUri: forward:/fallback
      discovery:
        locator:
          enabled: true
          lower-case-service-id: true
      default-filters:
      - name: Hystrix
        args:
          name: fallbackcmd
          fallbackUri: forward:/fallback
  application:
    name: gateway-server

hystrix:
  command:
    fallbackcmd:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 5000
#enable gateway endpoint
management:
  endpoint:
    gateway:
      enabled: true
  endpoints:
    web:
      exposure:
        include: gateway
```    
    
    
    
    
    