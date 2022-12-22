---
title: spring boot mybatis 运行分析2
author: ninuxGithub
layout: post
date: 2019-9-27 17:36:22
description: "spring boot mybatis 运行分析2"
tag: mybatis
---

### mybatis 自动配置执行
    MybatisAutoConfiguration进行了mybatis的自动配置
    SqlSessionFactoryBean 对象的创建
    设置数据源
    设置Configuration, 通过ConfigurationCustomizer接口可以进行configuration对象的自定义（配置属性的值）
    通过getObject()获取到SqlSessionFactory
    
    
    创建SqlSessionTemplate对象
    将SqlSessionFactory注入到该类中去
    
    进行mapper扫描
    扫描@Mapper注解，spring容器会对mapper组件进行扫描并且进行初始化，封装为mapper对象；
    
    思考问题： mapper包装成了什么对象？
    通过debug发现每个mapper接口都是属于MapperFactoryBean类型的， 我们需要进入到该类的里面进行源码的阅读
    在checkDaoConfig的时候configuration.addMapper 会调用MapperRegistry对mapper进行注册，将mapper放到了map里面
    使用class type 对应MapperProxyFactory形成对应关系
    getObject()会间接的调用MapperRegistry.getMapper  , 通过mapperProxyFactory.newInstance() 获取mapperProxy
    
    重点： 所以， mybatis所有的mapper的本质都是一个MapperProxy对象;
    
    
    
    控制器代用mapper方法的时候， 会通过对应的MapperProxy代理对象发起方法的调用， 通过代理将方法包装为MappedMethod对象
    然后进行执行目标方法；
    MappedMethod 判断方法的查询类型（select, delete, update,insert）-->sqlSession接口来进行执行
    SqlSessionTemplate来执行请求， 内部维护了一个sqlSessionProxy
    this.sqlSessionProxy = (SqlSession) newProxyInstance(
            SqlSessionFactory.class.getClassLoader(),
            new Class[] { SqlSession.class },
            new SqlSessionInterceptor());
    在执行selectXXX的时候会进入到SqlSessionInterceptor.invoke方法  获取sqlSession的实例对象（就是DefalutSqlSession对象）
    通过动态代理， 让DefalutSqlSession来执行我们的目标方法
    DefaultSqlSession实现了SqlSession接口所以会去请求这个类， 该类里面在会维护通过构造器传递进来的executor对象
    让方法通过executor来执行
    CacheExecutor 执行的时候加入了二级缓存
    BaseExecutor执行的时候加入了一级缓存
    然后交给SimpleExecutor进行执行 交给xxxStatement


### page helper 分析
   
    分页插件的代码

```java
@Configuration
@ConditionalOnBean(SqlSessionFactory.class)
@EnableConfigurationProperties(PageHelperProperties.class)
//需要MybatisAutoConfiguration加载完毕后， 再继续分页的自动配置
@AutoConfigureAfter(MybatisAutoConfiguration.class)
public class PageHelperAutoConfiguration {

    @Autowired
    private List<SqlSessionFactory> sqlSessionFactoryList;

    @Autowired
    private PageHelperProperties properties;

    /**
     * 接受分页插件额外的属性
     *
     * @return
     */
    @Bean
    @ConfigurationProperties(prefix = PageHelperProperties.PAGEHELPER_PREFIX)
    public Properties pageHelperProperties() {
        return new Properties();
    }

    @PostConstruct
    public void addPageInterceptor() {
        PageInterceptor interceptor = new PageInterceptor();
        Properties properties = new Properties();
        //先把一般方式配置的属性放进去
        properties.putAll(pageHelperProperties());
        //在把特殊配置放进去，由于close-conn 利用上面方式时，属性名就是 close-conn 而不是 closeConn，所以需要额外的一步
        properties.putAll(this.properties.getProperties());
        interceptor.setProperties(properties);
        for (SqlSessionFactory sqlSessionFactory : sqlSessionFactoryList) {
         
            //获取到configuration对象， 然后添加拦截器对象 ， 什么时候会被调用呢？
            //在MapperInterceptor 进行invoke的时候会创建sqlSession对象  ===> DefaultSqlSession对象 在这个过程中需要创建executor
            //newExecutor在生成executor的时候调用一次interceptorChain.pluginAll(executor);
            //newStatementHandler
            //newResultSetHandler
            //newParameterHandler
            //都会调用一次， 在查询的调用拦截器执行了分页的功能
            sqlSessionFactory.getConfiguration().addInterceptor(interceptor);
        }
    }

}
```   
    
    
    
    
    
    
    
    