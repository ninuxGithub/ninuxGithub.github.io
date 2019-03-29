---
title: mybatis 运行分析
author: ninuxGithub
layout: post
date: 2019-3-27 18:41:49
description: "mybatis 运行分析"
tag: java
---



### mybatis 运行过程


```java
MyBatisAutoConfiguration -->sqlSessionFactory（dataSource）方法会去创建一个SqlSessoinFactoryBean
SqlSessionFactoryBean 实现了 InitailizationBean  
-->启动的时候设置属性完毕会调用afterPropertiesSet
-->buildSqlSessionFactory() 受保护的方法： 在该方法内部创建XMLConfigBuilder对象， 
字面的意思就是xml配置的构建类，作用就是从resources 里面读我们的mapper.xml, 创建Configuration 对象


//MyBatisAutoConfiguration 的方法如下
    @Bean
    @ConditionalOnMissingBean
    public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
        SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
        ....
    }


	//XMLMapperBuilder 的方法如下
public void parse() {
	if (!configuration.isResourceLoaded(resource)) {
	  configurationElement(parser.evalNode("/mapper"));
	  configuration.addLoadedResource(resource);
	  bindMapperForNamespace(); ------> 创建mapper 根据命名空间
	}

	parsePendingResultMaps();
	parsePendingCacheRefs();
	parsePendingStatements();
}

private void bindMapperForNamespace() {
    String namespace = builderAssistant.getCurrentNamespace();
    if (namespace != null) {
      Class<?> boundType = null;
      try {
        boundType = Resources.classForName(namespace);
      } catch (ClassNotFoundException e) {
        //ignore, bound type is not required
      }
      if (boundType != null) {
        if (!configuration.hasMapper(boundType)) {
          // Spring may not know the real resource name so we set a flag
          // to prevent loading again this resource from the mapper interface
          // look at MapperAnnotationBuilder#loadXmlResource
          configuration.addLoadedResource("namespace:" + namespace);
          configuration.addMapper(boundType); ------>调用Configuration.addMapper------>MapperRegistry.addMapper
        }
      }
    }
  }



  //那么MapperRegistry 的逻辑
  public <T> void addMapper(Class<T> type) {
    if (type.isInterface()) {
      if (hasMapper(type)) {
        throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
      }
      boolean loadCompleted = false;
      try {
        knownMappers.put(type, new MapperProxyFactory<T>(type));---->MapperProxyFactory 会 new MapperProxy对象 
        // It's important that the type is added before the parser is run
        // otherwise the binding may automatically be attempted by the
        // mapper parser. If the type is already known, it won't try.
        MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);----->加载通过注解方式实现的mapper
        parser.parse();
        loadCompleted = true;
      } finally {
        if (!loadCompleted) {
          knownMappers.remove(type);
        }
      }
    }
  }


  //MapperProxyFactory代码如下
  @SuppressWarnings("unchecked")
  protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }

  public T newInstance(SqlSession sqlSession) {  	
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache); ----------->创建MapperProxy代理对象
    return newInstance(mapperProxy);
  }


  //Controller ----> xxxService.callMethod ---> xxxDao.callMethod(args)逻辑如下
  //MapperProxy 类如下  当service 方法调用dao层接口方法的时候， 通过调用了代理的invoke 方法去实现的---->继续往下就是SqlSessionTemplate
  //调用jdbc 底层的逻辑了
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
      } else if (isDefaultMethod(method)) {
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }

```



 
