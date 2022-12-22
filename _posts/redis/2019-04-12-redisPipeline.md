---
title: spring boot redisTemplate execute
author: ninuxGithub
layout: post
date: 2019-4-12 14:40:09
description: "spring boot redisTemplate execute"
tag: redis
---

### 目的
    分析
    redisTemplate.execute
    redisTemplate.executePipelined(RedisCallback<?> action)
    redisTemplate.executePipelined(SessionCallback<?> session)
   
    
```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = Application.class)
public class TestRedisTemplate {

    @Autowired
    RedisTemplate<String, String> redisTemplate;

    @Test
    public void testPipleLine() {
        /**
         *
         * 执行特定的一个操作可以指定是否暴露redis 连接， 另外也可以指定是否采用pipeline方式执行.
         * pipeline 执行的结果情况被丢弃了（为了符合只写的场景）
         *
         * Executes the given action object within a connection that can be exposed or not. Additionally, the connection can
         * be pipelined. Note the results of the pipeline are discarded (making it suitable for write-only scenarios).
         *
         * @param <T> return type
         * @param action callback object to execute
         * @param exposeConnection whether to enforce exposure of the native Redis Connection to callback code
         * @param pipeline whether to pipeline or not the connection for the execution
         * @return object returned by the action
         */
        //redisTemplate.execute(RedisCallback<T> action, boolean exposeConnection, boolean pipeline)

        Object result = redisTemplate.execute(new RedisCallback<Object>() {
            @Nullable
            @Override
            public Object doInRedis(RedisConnection connection) throws DataAccessException {
                RedisSerializer<String> serializer = redisTemplate.getStringSerializer();
                byte[] key = serializer.serialize("java");
                byte[] value = serializer.serialize("doc");
                connection.set(key, value);
                return null;
            }
        }, true, false);
        System.out.println(result); // 结果为null 说明执行的结果情况被丢弃了


        List<Object> list = redisTemplate.executePipelined(new RedisCallback<Object>() {

            @Nullable
            @Override
            public Object doInRedis(RedisConnection connection) throws DataAccessException {
                RedisSerializer<String> serializer = redisTemplate.getStringSerializer();
                Map<String, String> map = new HashMap<>();
                map.put("java", "doc");
                for (String saveKey : map.keySet()) {
                    byte[] key = serializer.serialize(saveKey);
                    byte[] value = serializer.serialize(map.get(saveKey));
                    connection.setEx(key, 10000, value);
                }
                return null;
            }
        });


       /**
       public interface RedisCallback<T> {
            //从redisTemplate中获取连接，不需要关注该链接对象是否开启或关闭了或者执行中抛出了异常
       
            //Gets called by {@link RedisTemplate} with an active Redis connection. Does not need to care about activating or
            //closing the connection or handling exceptions.
            @Nullable
            T doInRedis(RedisConnection connection) throws DataAccessException;
       }
       
       redisTemplate.executePipelined（RedisCallback<?> action）调用
       executePipelined(RedisCallback<?> action, @Nullable RedisSerializer<?> resultSerializer)
       
       
       @Override
        public List<Object> executePipelined(RedisCallback<?> action, @Nullable RedisSerializer<?> resultSerializer) {

            return execute((RedisCallback<List<Object>>) connection -> {
                //LettuceConnection.openPipeline 创建ppline = new ArrayList<>();
                connection.openPipeline();
                //标记性变量flag
                boolean pipelinedClosed = false;
                try {
                    //执行完action
                    Object result = action.doInRedis(connection);
                    //如果action 返回值不为null抛异常
                    if (result != null) {
                        throw new InvalidDataAccessApiUsageException(
                                "Callback cannot return a non-null value as it gets overwritten by the pipeline");
                    }
                    //关闭pipleline， resultHolder 里面的command 有执行的命令和执行的结果情况
                    List<Object> closePipeline = connection.closePipeline();
                    pipelinedClosed = true;
                    //获取运行的结果，有多少执行，返回执行的情况报告
                    return deserializeMixedResults(closePipeline, resultSerializer, hashKeySerializer, hashValueSerializer);
                } finally {
                    if (!pipelinedClosed) {
                        connection.closePipeline();
                    }
                }
            });
        }*/


        List<Object> list1 = null;
        try {
            list1 = redisTemplate.executePipelined(new RedisCallback<Object>() {

                @Nullable
                @Override
                public Object doInRedis(RedisConnection connection) throws DataAccessException {
                    RedisSerializer<String> serializer = redisTemplate.getStringSerializer();
                    Map<String, String> map = new HashMap<>();
                    map.put("javab", "doc");
                    for (String saveKey : map.keySet()) {
                        byte[] key = serializer.serialize(saveKey);
                        byte[] value = serializer.serialize(map.get(saveKey));
                        connection.setEx(key, 10000, value);
                    }
                    int a = 1, b = 0;

                    //加入异常，但是javab 依然被保存到了redis
                    //System.out.println(a / b);
                    return null;
                }
            });
        } catch (SerializationException e) {
            e.printStackTrace();
        }


        /*
        
        public interface SessionCallback<T> {
        
            // 使用同一个session执行内部的所有的操作
        	//Executes all the given operations inside the same session.
        	@Nullable
        	<K, V> T execute(RedisOperations<K, V> operations) throws DataAccessException;
        }
        
        */
        List<Object> list2 = null;
        try {
            list2 = redisTemplate.executePipelined(new SessionCallback<Object>() {
                @Nullable
                @Override
                public Object execute(RedisOperations operations) throws DataAccessException {
                    //开启事物
                    operations.multi();
                    operations.persist("java");

                    operations.opsForSet().add("bbbb", "1");
                    operations.opsForSet().add("bbbb", "2");
                    int a = 1, b = 0;
                    //抛异常，bbbb 没有被保存到redis
                    System.out.println(a / b);
                    operations.opsForSet().add("bbbb", "3");
                    //执行事物提交
                    operations.exec();
                    return null;
                }
            });
        } catch (Exception e) {
            e.printStackTrace();
        }
        List<Object> list3 = redisTemplate.executePipelined(new SessionCallback<Object>() {
            @Nullable
            @Override
            public Object execute(RedisOperations operations) throws DataAccessException {
                operations.persist("java");

                operations.opsForSet().add("ccc", "1");
                operations.opsForSet().add("ccc", "2");
                int a = 1, b = 0;
                System.out.println(a / b);
                //没有被保存到redis
                operations.opsForSet().add("ccc", "3");
                return null;
            }
        });
        System.out.println(list);
        System.out.println(list1);
        System.out.println(list2);
        System.out.println(list3);
    }

}

```  



### 总结
    execute 将执行的结果丢弃，可以指定是否暴露redis连接，是否开启pipeline
    
    executePipelined(RedisCallback<?> action) : 在redisCallback 里面可以写 ‘low level code’ ， redis 底层的命令
    因为在callback 函数里面传递了connection对象，可以参考RedisStringCommands.class , 可以执行 set, get, incr, decr
    命令被维护到pipeline (是一个list) ,可以获取执行的result情况；
    
    
    executePipelined(SessionCallback<?> session): 回调函数传递了一个RedisOperations 对象， 具体功能可以查看RedisOperations，提供了
    这意味RedisOperations可以以手动事物的方式提交数据
    
    思考：
    redisTemplate.execute(RedisCallback<T> action, false, true) vs  redisTemplate.executePipeline(RedisCallback<T> action)
    
    
    都是RedisCallback，采用pipeline 
    
    在RedisConnection 是否对外暴露，使用之前TransactionSynchronizationManager创建的连接
    
    exposeConnection ： false 创建新的连接

