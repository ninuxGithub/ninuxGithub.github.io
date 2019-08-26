---
title: spring boot mybatis 缓存
author: ninuxGithub
layout: post
date: 2019-8-26 19:34:04
description: "spring boot mybatis 缓存"
tag: mybatis
---

## mybatis 是如何开启一级， 二级缓存的

    使用过mybatis的都知道可以开启一级， 二级缓存来加快业务的查询的速度；
    mybatis一级缓存默认是开启的， 二级缓存是默认关闭的；
    
    查阅了一下资料
    一级缓存： 基于sqlSession的缓存， 可以跨多个mapper的， 因为一个sqlSession可以用来查询多个mapper的sql
    二级缓存： 基于mapper ，缓存的key对应的该mapper的namespace(命名空间)
    
    那么一级缓存是默认开启的在org.apache.ibatis.session.Configuration中的属性cacheEnabled属性来开启；
    可以通过mybatis.configuration.cache-enabled:true/false来修改
    
    那么二级缓存怎么开启呢？
    上也描述了二级缓存是基于mapper的，二级缓存对当前的命名空间的查询进行缓存
    需要在Mapper上面加入注解@CachedNamespace 我们可以查看源码MapperAnnotationBuilder 可以查看详细
    
    下面通源代码分析一级缓存指的是什么
    
    
    
```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = SpringBootStudyApplication.class)
public class MybatisTest {


    @Autowired
    ApplicationContext applicationContext;

    @Test
    public void test() {
        SqlSessionFactory sqlSessionFactory = applicationContext.getBean(SqlSessionFactory.class);
        SqlSession session = sqlSessionFactory.openSession();
        DBQueryMapper mapper = session.getMapper(DBQueryMapper.class);
        Student s1 = mapper.findById(1L);
        System.out.println(s1);
        Student s2 = mapper.findById(1L);
        System.out.println(s2);
        session.close();
    }

}


@Mapper
@Component

//xml 配置开启二级缓存 https://blog.csdn.net/u014749862/article/details/80297943
//spring boot 开启二级缓存 https://blog.csdn.net/j080624/article/details/80817225
//@CacheNamespace(flushInterval = 60000)
public interface DBQueryMapper {


    /**
     * 如果查询出来的为1就是交易日
     * 没有表
     *
     * @return
     */
//    @Select(value = "SELECT IfTradingDay from QT_TradingDayNew where  SecuMarket =83 and TradingDate= CONVERT(varchar(100), GETDATE(), 23);")
//    public int queryTradingDay();
    @SelectProvider(type = QueryProvider.class, method = "queryStudents")
    public List<Student> queryStudents();


    @Select("select * from student where id=#{id}")
    public Student findById(Long id);


    @Transactional
    @Insert("insert into student(id,name,age) values(#{id}, #{name}, #{age});")
    public void saveStudent(Student student);


    class QueryProvider {

        public String queryStudents() {
            return "select * from student;";
        }
    }

}
```    



    通过debug发现：  当我们使用同一个sqlSession来执行同一个对象的查询的时候
    sql会命中， 按道理会走缓存那么代码是如何处理的呢？  
    在BaseExecutor中有这么一段代码

```java
class BaseExecutor{
      public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
        ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
        if (closed) {
          throw new ExecutorException("Executor was closed.");
        }
        if (queryStack == 0 && ms.isFlushCacheRequired()) {
          clearLocalCache();
        }
        List<E> list;
        try {
          queryStack++;
          //如果resultHandler为空的时候 ，先查询localCache中是否有key 对应的值
          //如果有值那么我们不进行db 的query , 而是直接返回了缓存查询的结果
          //debug 的日志验证了这个论点；
          list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
          if (list != null) {
            handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
          } else {
            //查询db
            list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
          }
        } finally {
          queryStack--;
        }
        if (queryStack == 0) {
          for (DeferredLoad deferredLoad : deferredLoads) {
            deferredLoad.load();
          }
          // issue #601
          deferredLoads.clear();
          if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
            // issue #482
            clearLocalCache();
          }
        }
        return list;
      }
}
```    


    那么二级缓存是如何工作的呢？
    在DBQueryMapper类上面将@CacheNamespace注解打来， 那么就开启了二级缓存
    通过debug来调试发现
    
    
```java
class CachingExecutor{
     public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
          throws SQLException {
        //如果MappedStatement缓存不为空
        Cache cache = ms.getCache();
        if (cache != null) {
           //是否有必要刷新缓存
          flushCacheIfRequired(ms);
          if (ms.isUseCache() && resultHandler == null) {
            ensureNoOutParams(ms, boundSql);
            @SuppressWarnings("unchecked")
            //从缓存获取结果
            List<E> list = (List<E>) tcm.getObject(cache, key);
            if (list == null) {
                //query db
              list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
              //put to cache map
              tcm.putObject(cache, key, list); // issue #578 and #116
            }
            return list;
          }
        }
        //没开启二级缓存， 那么直接query db
        return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
      }
}
```    

    
## 总结
    扫描Mapper
    
    mybaits 启动的创建SqlSessionFactory 用来后续的sqlSession的创建
    创建sqlSessionTempalte模块， 来执行我们的sqlSession的查询功能了
    
    然后是开始创建Mapper对象了 ， 解析其中的注解， 封装为MappedStatement 保存到Configuration 的map里面去(key 为namespace)
    在调用我们的mapper里面的方法的时候  通过namespace 来获取MappedStatment
    MappedStatment 里面有...

```class
public final class MappedStatement {

  private String resource;
  private Configuration configuration;
  private String id;
  private Integer fetchSize;
  private Integer timeout;
  private StatementType statementType;
  private ResultSetType resultSetType;
  private SqlSource sqlSource;
  private Cache cache;
  private ParameterMap parameterMap;
  private List<ResultMap> resultMaps;
  private boolean flushCacheRequired;
  private boolean useCache;
  private boolean resultOrdered;
  private SqlCommandType sqlCommandType;
  private KeyGenerator keyGenerator;
  private String[] keyProperties;
  private String[] keyColumns;
  private boolean hasNestedResultMaps;
  private String databaseId;
  private Log statementLog;
  private LanguageDriver lang;
  private String[] resultSets;
  }
```
    是我们Mapper运行的时候的是依赖的重要对象；

    二级缓存是依赖依赖一级缓存的
    缓存的优先级是  二级缓存  >  一级缓存
    如果二级缓存有了就不查询一级缓存
    如果二级缓存缓存没有才有可能去查询一级缓存   --> （这个没有确切的验证）， 但是如果二级缓存没有开启是直接查询一级缓存的
    如果没有查询到， 那么直接查询db
    
    
    在执行mybatis查询的时候， 在创建SqlSession时候需要传递一个Executor 我们可以查询一下DefaultSqlSession的构造器
    DefaultSqlSession(Configuration configuration, Executor executor, boolean autoCommit)
    Executor executor = configuration.newExecutor(tx, execType);
    
    原理executor是configuration对象来创建的， 是根据ExecutorType 来创建不同的执行器
    执行器有多种， 可以查看接口org.apache.ibatis.executor.Executor 的实现类
    不同的执行器有不同的特点
    
    通过上面的debug发现  如果开启了二级缓存的时候需要经过CacheingExecutor -->BaseExecutor -->SimpleExecutor
    如果没有开启二级缓存， 那么就没有结果CachingExecutor, 
    所以二级缓存是的业务是有CachingExecutor来执行的
    query db 的业务交给后续的Executor;
    
        