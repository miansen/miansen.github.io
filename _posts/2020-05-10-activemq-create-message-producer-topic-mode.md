---
layout: post
title: 六、学习ActiveMQ-消息生产者（主题模式）
date: 2020-05-10
categories: ActiveMQ
tags: ActiveMQ
author: 龙德
---

* content
{:toc}

主题模式和队列模式非常类似，只需要将 createQueue 改成 createTopic 就行了。

下面是代码：

```java
public class TopicProducer {

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
            session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);

            // 5.使用会话对象创建目标对象，这里我们创建的是 topic，也就是一对多模式
            Destination destination = session.createTopic("topic-test-01");

            // 6.使用会话对象创建生产者对象
            producer = session.createProducer(destination);

            // 7.使用会话对象创建一个文本消息对象
            TextMessage textMessage = session.createTextMessage();

            String text = "Hello ActiveMQ - " + new Date().getTime();

            // 8.设置消息内容
            textMessage.setText(text);

            // 9.发送消息
            producer.send(textMessage);
            System.out.println("已生产消息：" + text);

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

主题模式下，ActiveMQ 是不保存消息的，它是无状态的。假如无人订阅，那么生产出来的消息就是一条废消息，所以，一般情况下，需要先启动消费者，再启动生产者。

上述内容可以联系微信里的关注公众号来理解，我关注了公众号之后，公众号会给我推送消息，我没有关注，自然不会收到微信公众号发来的消息。并且，这个推送是自我关注之后，才收到的，之前微信公众号发布过的消息，不会给我补发和补推送。