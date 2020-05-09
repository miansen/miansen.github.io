---
layout: post
title: 三、学习ActiveMQ-消息生产者（队列模式）
date: 2020-05-07
categories: ActiveMQ
tags: ActiveMQ
author: 龙德
---

* content
{:toc}

ActiveMQ 消息传送有两种模型：

- 点对点（P2P），即队列模式
- 发布订阅（PUB/SUB）模式，即主题模式

队列模式的特点：

1.生产者将消息发布到队列中，每个消息只能被一个消费者消费，属于一对一的关系。

2.生产者和消费者不存在时间上的相关性，先启动生产者还是消费者都是一样的。

3.生产者生产时，队列模式默认是会持久化消息的，所以先启动生产者，再启动消费者，消费者还是能消费到消息，也就是说队列中的消息不会丢弃。

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
public class QueueProducer {

    public static void main(String[] args) {
        
        Connection connection = null;
        Session session = null;
        MessageProducer producer = null;
        
        try {
            // 1.创建连接工厂，需要传入ip和端口号，这里我们使用默认的
            ActiveMQConnectionFactory connectionFactory = new ActiveMQConnectionFactory(
                    ActiveMQConnectionFactory.DEFAULT_BROKER_BIND_URL);
            
            // 2.使用连接工厂创建一个连接对象
            connection = connectionFactory.createConnection();
            
            // 3.开启连接
            connection.start();
            
            // 4.使用连接对象创建会话对象。
            // 这里有两个参数：第一个参数是开启事务，第二个参数表示签收模式，后面再详细介绍。
            session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
            
            // 5.使用会话对象创建目标对象，这里我们创建的是 queue，也就是点对点的模式。除此之外还有一对多模式：createTopic()
            Destination destination = session.createQueue("queue-test-01");
            
            // 6.使用会话对象创建生产者对象
            producer = session.createProducer(destination);
            
            // 7.使用会话对象创建一个文本消息对象
            TextMessage textMessage = session.createTextMessage();
            
            String text = "Hello ActiveMQ - " + new Date().getTime();
            
            // 8.设置消息内容
            textMessage.setText(text);
            
            // 9.发送消息
            producer.send(textMessage);
            
            System.out.println("已发送消息：" + text);
            
        } catch (JMSException e) {
            e.printStackTrace();
        } finally {
            // 10.关闭资源
            if (producer != null) {
                try {
                    producer.close();
                } catch (JMSException e) {
                    e.printStackTrace();
                }
            }
            if (session != null) {
                try {
                    session.close();
                } catch (JMSException e) {
                    e.printStackTrace();
                }
            }
            if (connection != null) {
                try {
                    connection.close();
                } catch (JMSException e) {
                    e.printStackTrace();
                }
            }
        }
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