---
layout: post
title: 三、学习ActiveMQ-创建消息生产者（Queue模式）
date: 2020-05-07
categories: ActiveMQ
tags: ActiveMQ
author: 龙德
---

* content
{:toc}

在 Java 中使用 ActiveMQ，跟使用 JDBC、JPA 一样，都是有套路的。下面我们通过一个 Hello World 级别的例子，来感受一下在 Java 中如何使用 ActiveMQ。

先引入依赖

```xml
<dependency>
  <groupId>org.apache.activemq</groupId>
  <artifactId>activemq-core</artifactId>
  <version>5.7.0</version>
</dependency>
```

```java
public class Producer {

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
        
        // 5.使用会话对象创建目标对象，这里我们创建的是 queue，也就是点对点的模式。除此之外还有一对多模式：createTopic()
        Destination destination = session.createQueue("queue-test-01");
        
        // 6.使用会话对象创建生产者对象
        MessageProducer producer = session.createProducer(destination);
        
        // 7.使用会话对象创建一个文本消息对象
        TextMessage textMessage = session.createTextMessage();
        
        // 8.设置消息内容
        textMessage.setText("Hello ActiveMQ");
        
        // 9.发送消息
        producer.send(textMessage);
        
        // 10.关闭资源
        producer.close();
        session.close();
        connection.close();
    
    }
}
```

运行生产者，查看控制台，点击 "Queues"，可以看到生产了 1 条消息，有 1 条消息等待消费。

![image](https://miansen.wang/assets/20200507154718.png)

上面的表头分别是：

- Name：队列名称
- Messages Enqueued：生产条数，记做 P
- Messages Dequeued：消费条数，记做 C
- Number Of Consumers：消费者数量
- Number Of Pending Messages：表示还有多少条消息没有被消费，实际上是表示消息的积压程度，就是 P-C