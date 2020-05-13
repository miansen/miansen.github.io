---
layout: post
title: 7.学习ActiveMQ-消息消费者（主题模式）
date: 2020-05-11
categories: ActiveMQ
tags: ActiveMQ
author: 龙德
---

* content
{:toc}

主题模式和队列模式非常类似，只需要将 createQueue 改成 createTopic 就行了。

下面是代码：

```java
public class TopicConsumer {

    public static void main(String[] args) throws JMSException {

        // 1.创建连接工厂，需要传入ip和端口号，这里我们使用默认的
        ActiveMQConnectionFactory connectionFactory = new ActiveMQConnectionFactory(
                ActiveMQConnectionFactory.DEFAULT_BROKER_BIND_URL);

        // 2.使用连接工厂创建一个连接对象
        Connection connection = connectionFactory.createConnection();

        // 3.开启连接
        connection.start();

        // 4.使用连接对象创建会话对象。
        Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);

        // 5.使用会话对象创建目标对象，这里我们创建的是主题模式，生产者与消费者的名称要保持一致。
        Destination destination = session.createTopic("topic-test-01");

        // 6.使用会话对象创建消费者对象
        MessageConsumer consumer = session.createConsumer(destination);

        // 7.通过监听器的方式来接收消息
        consumer.setMessageListener(new MessageListener() {

            @Override
            public void onMessage(Message message) {
                try {
                    if (message instanceof TextMessage) {
                        System.out.println("已消费消息：" + ((TextMessage) message).getText());
                    }
                } catch (JMSException e) {
                    e.printStackTrace();
                }
            }
        });
    }
}
```

先启动消费者，为了看清1对多的效果，我们启动3个消费者。

![image](https://miansen.wang/assets/20200511105241.png)

可以看到，名为 "topic-test-01" 的主题有 3 个订阅者。

接着我们启动生产者，生产1条消息。就可以看到3个消费者都消费到了消息。

![image](https://miansen.wang/assets/20200511110604.png)