---
layout: post
title: 四、学习ActiveMQ-创建消息消费者（Queue模式）
date: 2020-05-07
categories: ActiveMQ
tags: ActiveMQ
author: 龙德
---

* content
{:toc}

创建消息消费者的套路也跟生产者差不多

```java
public class Consumer {

    public static void main(String[] args) throws JMSException {
    
        // 1.创建连接工厂，需要传入ip和端口号，这里我们使用默认的
        ActiveMQConnectionFactory connectionFactory = new ActiveMQConnectionFactory(
                ActiveMQConnectionFactory.DEFAULT_BROKER_BIND_URL);
        
        // 2.使用连接工厂创建一个连接对象
        Connection connection = connectionFactory.createConnection();
        
        // 3.开启连接
        connection.start();
        
        // 4.使用连接对象创建会话对象。
        // 这里有两个参数：第一个参数是开启事务，第二个参数表示签收模式，后面再详细介绍。
        Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
        
        // 5.使用会话对象创建目标对象，这里我们创建的是 queue，也就是点对点的模式。与生产者保持一致，并且名称也要一致。
        Destination destination = session.createQueue("queue-test-01");
        
        // 6.使用会话对象创建消费者对象
        MessageConsumer consumer = session.createConsumer(destination);
        
        // 7.使用 receive() 方法消费消息
        TextMessage msg;
        while (true) {
            // receive() 方法当没有消息的时候会阻塞在这，等待提供者提供消息，后面介绍使用监听的方式来获取消息
            msg = (TextMessage) consumer.receive();
            System.out.println("消费消息：" + msg.getText());
        }
    }
}
```

运行消费者，查看控制台，可以看到消费了 1 条消息，有 0 条消息等待消费。

![image](https://miansen.wang/assets/20200507160537.png)