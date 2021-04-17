---
title: rabbitmq dead letter
author: ninuxGithub
layout: post
date: 2021-4-17 20:33:29
description: "rabbitmq 死信队列"
tag: rabbitmq
---



### 死信队列
    1.队列nack, reject, 并且requeue = false
    2.消息队列的长度达到了饱和， 已满了
    3.消息的ttl 时间到了
    

    被接受的队列拒绝了
    
```java
public class Consumer {


    public static void main(String[] args) throws IOException {
        Connection connection = ConnectionFactory.getConnection();

        Channel channel = connection.createChannel();
        String dlxQueueName = "dlx.queue.nack";
        String queueName = "test_queue.nack";

        //控制消费者每次只消费1个消息， 抓取消息的最大数
        channel.basicQos(1);

        /*监听死信队列*/
        channel.basicConsume(dlxQueueName, false, new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                String json = new String(body);
                Message message = JSONUtil.toBean(json, Message.class);
                Date date = new Date();

                long between = DateUtil.between(message.getTimestamp(), date, DateUnit.SECOND);
                String format = String.format("[%s] Received message:%s ; spend seconds: %d s ", dlxQueueName, message.toString(), between);
                System.out.println(format);
                channel.basicAck(envelope.getDeliveryTag(), false);
            }
        });

        /*监听目标队列*/
        channel.basicConsume(queueName, false, new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                System.err.println("拒收消息");
                channel.basicNack(envelope.getDeliveryTag(), false, false);
            }
        });


    }
}
```


    消息队列已满了
    
```java
public class Consumer {


    public static void main(String[] args) throws IOException {
        Connection connection = ConnectionFactory.getConnection();

        Channel channel = connection.createChannel();
        String dlxQueueName = "dlx.queue.length";
        String queueName = "test_queue.length";

        //控制消费者每次只消费1个消息， 抓取消息的最大数
        channel.basicQos(1);

        /*监听死信队列*/
        channel.basicConsume(dlxQueueName, false, new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                String json = new String(body);
                Message message = JSONUtil.toBean(json, Message.class);
                Date date = new Date();

                long between = DateUtil.between(message.getTimestamp(), date, DateUnit.SECOND);
                String format = String.format("[%s] Received message:%s ; spend seconds: %d s ", dlxQueueName, message.toString(), between);
                System.out.println(format);
                channel.basicAck(envelope.getDeliveryTag(), false);
            }
        });

        /*监听目标队列*/
        channel.basicConsume(queueName, false, new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                String json = new String(body);
                Message message = JSONUtil.toBean(json, Message.class);
                Date date = new Date();

                long between = DateUtil.between(message.getTimestamp(), date, DateUnit.SECOND);
                System.out.println(String.format("[%s] Received message:%s ; spend seconds: %d s ", queueName, message.toString(), between));

                channel.basicAck(envelope.getDeliveryTag(), false);
            }
        });


    }
}
```


  第三种情况就是  ttl 时间到了
 
 
```java
public class Consumer {


    public static void main(String[] args) throws IOException {
        Connection connection = ConnectionFactory.getConnection();

        Channel channel = connection.createChannel();
        String dlxQueueName = "dlx.queue";
        String queueName = "test_queue";

        //控制消费者每次只消费1个消息， 抓取消息的最大数
        channel.basicQos(1);

        /*监听死信队列*/
        channel.basicConsume(dlxQueueName, false, new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                String json = new String(body);
                Message message = JSONUtil.toBean(json, Message.class);
                Date date = new Date();

                long between = DateUtil.between(message.getTimestamp(), date, DateUnit.SECOND);
                String format = String.format("[%s] Received message:%s ; spend seconds: %d s ", dlxQueueName, message.toString(), between);
                System.out.println(format);
                channel.basicAck(envelope.getDeliveryTag(), false);
            }
        });

        /*监听目标队列*/
        channel.basicConsume(queueName, false, new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                String json = new String(body);
                Message message = JSONUtil.toBean(json, Message.class);
                Date date = new Date();

                //睡眠5秒收到1条消息， 然后在30s 内会收到6条消息的投递，其他的消息会被死信队列消费掉
                try {
                    Thread.sleep(5_000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                long between = DateUtil.between(message.getTimestamp(), date, DateUnit.SECOND);
                System.out.println(String.format("[%s] Received message:%s ; spend seconds: %d s ", queueName, message.toString(), between));

                channel.basicAck(envelope.getDeliveryTag(), false);
            }
        });
    }
}

public class Producer {


    // # 匹配多个词
    //* 匹配一个词

    public static void main(String[] args) throws IOException, TimeoutException {
        Connection connection = ConnectionFactory.getConnection();

        String dlxQueueName = "dlx.queue.nack";
        String dlxExchangeName = "dlx.exchange.nack";
        String dlxRoutingKey = "topic.#";
        Channel channel = connection.createChannel();

        String exchangeName = "test_exchange.nack";
        String queueName = "test_queue.nack";
        String routingKey = "topic.item3";
        Map<String, Object> arguments = new HashMap<>(16);
        arguments.put("x-dead-letter-exchange", dlxExchangeName);
        channel.exchangeDeclare(exchangeName, "topic", true, false, null);
        channel.queueDeclare(queueName, true, false, false, arguments);
        channel.queueBind(queueName, exchangeName, routingKey);


        //dlx queue config
        channel.exchangeDeclare(dlxExchangeName, "topic", true, false, null);
        channel.queueDeclare(dlxQueueName, true, false, false, null);
        channel.queueBind(dlxQueueName, dlxExchangeName, dlxRoutingKey);

        for (int i=0; i<10; i++){

            Message message = new Message();
            message.setMsg("this is a message test ttl");
            message.setTimestamp(new Date());

            channel.basicPublish(exchangeName, routingKey, MessageProperties.PERSISTENT_TEXT_PLAIN, JSONUtil.toJsonStr(message).getBytes());
        }
        System.out.println("==>已经发送消息");
        channel.close();
        connection.close();
    }
}



public class Producer {


    // # 匹配多个词
    //* 匹配一个词

    public static void main(String[] args) throws IOException, TimeoutException {
        Connection connection = ConnectionFactory.getConnection();

        String dlxQueueName = "dlx.queue.length";
        String dlxExchangeName = "dlx.exchange.length";
        String dlxRoutingKey = "topic.#";
        Channel channel = connection.createChannel();

        String exchangeName = "test_exchange.length";
        String queueName = "test_queue.length";
        String routingKey = "topic.item2";
        Map<String, Object> arguments = new HashMap<>(16);
        arguments.put("x-dead-letter-exchange", dlxExchangeName);
        //30s time to live 存活时间
        arguments.put("x-max-length", 5);
        channel.exchangeDeclare(exchangeName, "topic", true, false, null);
        channel.queueDeclare(queueName, true, false, false, arguments);
        channel.queueBind(queueName, exchangeName, routingKey);


        //dlx queue config
        channel.exchangeDeclare(dlxExchangeName, "topic", true, false, null);
        channel.queueDeclare(dlxQueueName, true, false, false, null);
        channel.queueBind(dlxQueueName, dlxExchangeName, dlxRoutingKey);

        for (int i=0; i<10; i++){

            Message message = new Message();
            message.setMsg("this is a message test ttl");
            message.setTimestamp(new Date());

            channel.basicPublish(exchangeName, routingKey, MessageProperties.PERSISTENT_TEXT_PLAIN, JSONUtil.toJsonStr(message).getBytes());
        }
        System.out.println("==>已经发送消息");
        channel.close();
        connection.close();
    }
}



public class Producer {


    // # 匹配多个词
    //* 匹配一个词

    public static void main(String[] args) throws IOException, TimeoutException {
        Connection connection = ConnectionFactory.getConnection();

        String dlxQueueName = "dlx.queue";
        String dlxExchangeName = "dlx.exchange";
        String dlxRoutingKey = "topic.#";
        Channel channel = connection.createChannel();

        String exchangeName = "test_exchange";
        String queueName = "test_queue";
        String routingKey = "topic.item";
        Map<String, Object> arguments = new HashMap<>(16);
        arguments.put("x-dead-letter-exchange", dlxExchangeName);
        //30s time to live 存活时间
        arguments.put("x-message-ttl", 30_000);
        channel.exchangeDeclare(exchangeName, "topic", true, false, null);
        channel.queueDeclare(queueName, true, false, false, arguments);
        channel.queueBind(queueName, exchangeName, routingKey);


        //dlx queue config
        channel.exchangeDeclare(dlxExchangeName, "topic", true, false, null);
        channel.queueDeclare(dlxQueueName, true, false, false, null);
        channel.queueBind(dlxQueueName, dlxExchangeName, dlxRoutingKey);

        for (int i=0; i<10; i++){

            Message message = new Message();
            message.setMsg("this is a message test ttl");
            message.setTimestamp(new Date());

            channel.basicPublish(exchangeName, routingKey, MessageProperties.PERSISTENT_TEXT_PLAIN, JSONUtil.toJsonStr(message).getBytes());
        }
        System.out.println("==>已经发送消息");
        channel.close();
        connection.close();
    }
}


@Data
public class Message {

    private String msg;

    private Date timestamp;


    @Override
    public String toString() {
        final StringBuffer sb = new StringBuffer("Message{");
        sb.append("msg='").append(msg).append('\'');
        sb.append(", timestamp=").append(DateUtil.format(timestamp, DatePattern.NORM_DATETIME_PATTERN));
        sb.append('}');
        return sb.toString();
    }
}

```

   总结一下整理成demo

   参考了： https://www.cnblogs.com/yangk1996/p/12674015.html



   
        

    