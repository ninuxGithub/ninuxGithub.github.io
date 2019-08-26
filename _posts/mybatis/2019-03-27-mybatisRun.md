---
title: spring boot mybatis 运行分析
author: ninuxGithub
layout: post
date: 2019-3-27 18:41:49
description: "spring boot mybatis 运行分析"
tag: mybatis
---



### spring boot mybatis 运行过程
    MyBatisAutoConfiguration -->sqlSessionFactory（dataSource）方法会去创建一个SqlSessoinFactoryBean
    SqlSessionFactoryBean  -->afterPropertiesSet 
                           -->buildSqlSessionFactory 
                                    -->xmlConfigBuilder.parse()解析mybatis Config
                                    -->xmlMapperBuilder.parse()解析mybatis XML Mapper
                                    -->this.sqlSessionFactoryBuilder.build(configuration) 创建一个SqlSessionFactory对象返回
    
    XMLMapperBuilder -->   parse() 
                                -->   bindMapperForNamespace() 创建namespace底下的mapper            
    
    //扫描application中加入了@Mapper注解的bean, 通过spring boot 启动的时候来创建bean的实例对象
    //扫描@CacheNamespace 来确定是否开启二级缓存（默认是开启一级缓存的）
    //扫描方法上面的@Options, @SelectKey, @ResultMap封装为MappedStatement对象
    //然后add到Configuration.mappedStatements的map里面去（key为mapper的spacename）
    MapperRegistry --> addMapper()
                        -->new MapperProxyFactory(type)  jdk 动态代理   
                            -->new MapperProxy<T>(sqlSession, mapperInterface, methodCache)
                                -->invoke 目标方法代理
                                    -->mapperMethod.execute(sqlSession, args)
    
    //前端调用controller-->调用xxxMapper接口的方法的时候， 目标方法通过MapperProxy结果一次代理
    //封装为MapperMethod对象 继续执行
    MapperMethod -- >  execute() 判断执行查询，插入，删除，还是修改    
    
    SqlSessionTemplate --select()
                            -->sqlSessionProxy.select 通过代理来select
                                   -->代理类SqlSessionInterceptor invoke 开启查询 jdk 动态代理
    如何代理的呢？
    method.invoke(sqlSession, args); 就是sqlSession 对象发起一个查询动作， 调用selectList 方法                          
    spring boot 回去调用 DefaultSqlSession  , DefaultSqlSession 是SqlSession的默认的实现类
    DefaultSqlSession.selectList()
        --->CachingExecutor.query()
        ---> BaseExecutor.query()
        ---> BaseExecutor.QueryFromDataBase()
        ---> SimpleExecutor.doQuery()
        ---> RoutingStatmentHandler.query()
        ---> PreparedStatementHandler.query()
    
    
    那么SqlSession 是如何获取的呢?
    可以看看SqlSessionUtils 通过holder的思想将sqlSession维护起来
    
    
    org.apache.ibatis.session.Configuration 是一个中央控制器，负责控制和协调mybatis的bean的创建和调用


调用的java类如下
                    

```java
class TestApp{
    @Test
    public void testQuery(){
        cacheDataService.findSecuMainList();
    }
}

@Configuration
@ConditionalOnClass({SqlSessionFactory.class, SqlSessionFactoryBean.class})
@ConditionalOnBean({DataSource.class})   //等待数据源对象建立以后再创建MybatisAutoConfiguration
@EnableConfigurationProperties({MybatisProperties.class})
@AutoConfigureAfter({DataSourceAutoConfiguration.class})
public class MybatisAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean
    public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
        SqlSessionFactoryBean factory = new SqlSessionFactoryBean();//创建一个SqlSessionFactoryBean
        factory.setDataSource(dataSource);//设置数据源
        factory.setVfs(SpringBootVFS.class);
        if (StringUtils.hasText(this.properties.getConfigLocation())) {
            factory.setConfigLocation(this.resourceLoader.getResource(this.properties.getConfigLocation()));
        }

        org.apache.ibatis.session.Configuration configuration = this.properties.getConfiguration();
        if (configuration == null && !StringUtils.hasText(this.properties.getConfigLocation())) {
            configuration = new org.apache.ibatis.session.Configuration();
        }

        if (configuration != null && !CollectionUtils.isEmpty(this.configurationCustomizers)) {
            Iterator var4 = this.configurationCustomizers.iterator();

            while(var4.hasNext()) {
                ConfigurationCustomizer customizer = (ConfigurationCustomizer)var4.next();
                customizer.customize(configuration);
            }
        }

        factory.setConfiguration(configuration);
        
        //......

        return factory.getObject();
    }

}


//实现了InitializingBean ， 重写afterPropertiesSet方法， 设置属性完毕后调用afterPropertiesSet

public class SqlSessionFactoryBean implements FactoryBean<SqlSessionFactory>, InitializingBean, ApplicationListener<ApplicationEvent> {
    

    public void afterPropertiesSet() throws Exception {
        Assert.notNull(this.dataSource, "Property 'dataSource' is required");
        Assert.notNull(this.sqlSessionFactoryBuilder, "Property 'sqlSessionFactoryBuilder' is required");
        Assert.state(this.configuration == null && this.configLocation == null || this.configuration == null || this.configLocation == null, "Property 'configuration' and 'configLocation' can not specified with together");
        this.sqlSessionFactory = this.buildSqlSessionFactory();
    }

    protected SqlSessionFactory buildSqlSessionFactory() throws IOException {
        XMLConfigBuilder xmlConfigBuilder = null;
        Configuration configuration;
        if (this.configuration != null) {
            configuration = this.configuration;
            if (configuration.getVariables() == null) {
                configuration.setVariables(this.configurationProperties);
            } else if (this.configurationProperties != null) {
                configuration.getVariables().putAll(this.configurationProperties);
            }
        } else if (this.configLocation != null) {
            xmlConfigBuilder = new XMLConfigBuilder(this.configLocation.getInputStream(), (String)null, this.configurationProperties);
            configuration = xmlConfigBuilder.getConfiguration();
        } else {
            if (LOGGER.isDebugEnabled()) {
                LOGGER.debug("Property 'configuration' or 'configLocation' not specified, using default MyBatis Configuration");
            }

            configuration = new Configuration();
            if (this.configurationProperties != null) {
                configuration.setVariables(this.configurationProperties);
            }
        }
        
        //.....略

        configuration.setEnvironment(new Environment(this.environment, this.transactionFactory, this.dataSource));
        if (!ObjectUtils.isEmpty(this.mapperLocations)) {
            Resource[] var29 = this.mapperLocations;
            var27 = var29.length;

            for(var5 = 0; var5 < var27; ++var5) {
                Resource mapperLocation = var29[var5];
                if (mapperLocation != null) {
                    try {
                        
                        //通过XMLMapperBuilder 去解析 xxxxmapper.xml
                        XMLMapperBuilder xmlMapperBuilder = new XMLMapperBuilder(mapperLocation.getInputStream(), configuration, mapperLocation.toString(), configuration.getSqlFragments());
                        xmlMapperBuilder.parse();
                    } catch (Exception var20) {
                        throw new NestedIOException("Failed to parse mapping resource: '" + mapperLocation + "'", var20);
                    } finally {
                        ErrorContext.instance().reset();
                    }

                    if (LOGGER.isDebugEnabled()) {
                        LOGGER.debug("Parsed mapper file: '" + mapperLocation + "'");
                    }
                }
            }
        } else if (LOGGER.isDebugEnabled()) {
            LOGGER.debug("Property 'mapperLocations' was not specified or no matching resources found");
        }

        return this.sqlSessionFactoryBuilder.build(configuration);
    }
    
}



public class XMLMapperBuilder extends BaseBuilder {

    //开始解析xml
    public void parse() {
        if (!this.configuration.isResourceLoaded(this.resource)) {
            this.configurationElement(this.parser.evalNode("/mapper"));
            this.configuration.addLoadedResource(this.resource);
            this.bindMapperForNamespace(); 
        }

        this.parsePendingResultMaps();
        this.parsePendingCacheRefs();
        this.parsePendingStatements();
    }

    

    private void bindMapperForNamespace() {
        String namespace = this.builderAssistant.getCurrentNamespace();
        if (namespace != null) {
            Class boundType = null;

            try {
                boundType = Resources.classForName(namespace);
            } catch (ClassNotFoundException var4) {
                ;
            }

            if (boundType != null && !this.configuration.hasMapper(boundType)) {
                this.configuration.addLoadedResource("namespace:" + namespace);
                
                //将解析的Mapper 添加到 configuration对象
                //调用Configuration.addMapper------>MapperRegistry.addMapper
                this.configuration.addMapper(boundType);
            }
        }

    }
}



public class MapperRegistry {   

    public <T> void addMapper(Class<T> type) {
        if (type.isInterface()) {
            if (this.hasMapper(type)) {
                throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
            }

            boolean loadCompleted = false;

            try {
                //创建MapperProxyFactory对象
                this.knownMappers.put(type, new MapperProxyFactory(type));
                MapperAnnotationBuilder parser = new MapperAnnotationBuilder(this.config, type);
                parser.parse();
                loadCompleted = true;
            } finally {
                if (!loadCompleted) {
                    this.knownMappers.remove(type);
                }

            }
        }

    }
  
}

public class MapperProxyFactory<T> {

  private final Class<T> mapperInterface;
  private final Map<Method, MapperMethod> methodCache = new ConcurrentHashMap<Method, MapperMethod>();

  public MapperProxyFactory(Class<T> mapperInterface) {
    this.mapperInterface = mapperInterface;
  }

  public Class<T> getMapperInterface() {
    return mapperInterface;
  }

  public Map<Method, MapperMethod> getMethodCache() {
    return methodCache;
  }

  @SuppressWarnings("unchecked")
  protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }

  public T newInstance(SqlSession sqlSession) {
    //MapperProxy对象被创建
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }

}


//Controller ----> xxxService.callMethod ---> xxxDao.callMethod(args)逻辑如下
  //MapperProxy 类如下  当service 方法调用dao层接口方法的时候， 通过调用了代理的invoke 方法去实现的---->继续往下就是SqlSessionTemplate
  //调用jdbc 底层的逻辑了
public class MapperProxy<T> implements InvocationHandler, Serializable {

  @Override
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
}




public class SqlSessionTemplate implements SqlSession, DisposableBean {
    

    public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType, PersistenceExceptionTranslator exceptionTranslator) {
        Assert.notNull(sqlSessionFactory, "Property 'sqlSessionFactory' is required");
        Assert.notNull(executorType, "Property 'executorType' is required");
        this.sqlSessionFactory = sqlSessionFactory;
        this.executorType = executorType;
        this.exceptionTranslator = exceptionTranslator;
        //sqlSessionProxy 是一个代理 内部类SqlSessionInterceptor 
        this.sqlSessionProxy = (SqlSession)Proxy.newProxyInstance(SqlSessionFactory.class.getClassLoader(), new Class[]{SqlSession.class}, new SqlSessionTemplate.SqlSessionInterceptor());
    }

    

    public void select(String statement, ResultHandler handler) {
        //通过代理来实现 select 
        this.sqlSessionProxy.select(statement, handler);
    }

    

    private class SqlSessionInterceptor implements InvocationHandler {
        private SqlSessionInterceptor() {
        }

        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            //获取Sqlsession
            SqlSession sqlSession = SqlSessionUtils.getSqlSession(SqlSessionTemplate.this.sqlSessionFactory, SqlSessionTemplate.this.executorType, SqlSessionTemplate.this.exceptionTranslator);

            Object unwrapped;
            try {
                //invoke 方法
                Object result = method.invoke(sqlSession, args);
                if (!SqlSessionUtils.isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
                    //提交事物
                    sqlSession.commit(true);
                }

                unwrapped = result;
            } catch (Throwable var11) {
                unwrapped = ExceptionUtil.unwrapThrowable(var11);
                if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
                    SqlSessionUtils.closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
                    sqlSession = null;
                    Throwable translated = SqlSessionTemplate.this.exceptionTranslator.translateExceptionIfPossible((PersistenceException)unwrapped);
                    if (translated != null) {
                        unwrapped = translated;
                    }
                }

                throw (Throwable)unwrapped;
            } finally {
                if (sqlSession != null) {
                    //关闭SqlSession
                    SqlSessionUtils.closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
                }

            }

            return unwrapped;
        }
    }
}



```



 
