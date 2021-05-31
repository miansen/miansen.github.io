---
layout: post
title: 14.学习ActiveMQ-主题的持久化
date: 2020-05-12
categories: 消息队列
tags: ActiveMQ
author: 龙德
---

* content
{:toc}

主题的持久化跟队列有点不一样，我们先看生产者。

生产者改动不大，只需要设置 producer.setDeliveryMode(DeliveryMode.PERSISTENT) 就可以了。

以下是代码：

```java
public class DurableTopicProducer {

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
            
            producer.setDeliveryMode(DeliveryMode.PERSISTENT);
            
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

然后是消费者，消费者改动比较多，以下是代码：

```java
public class DurableTopicConsumer {

    public static void main(String[] args) throws JMSException {

        // 1.创建连接工厂，需要传入ip和端口号，这里我们使用默认的
        ActiveMQConnectionFactory connectionFactory = new ActiveMQConnectionFactory(
                ActiveMQConnectionFactory.DEFAULT_BROKER_BIND_URL);

        // 2.使用连接工厂创建一个连接对象
        Connection connection = connectionFactory.createConnection();
        
        // 3.设置ClientID
        
        connection.setClientID("client1");

        // 4.开启连接
        connection.start();

        // 5.使用连接对象创建会话对象
        Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);

        // 6.使用会话对象创建目的地对象，生产者与消费者的名称要保持一致。
        Topic topic = session.createTopic("topic-test-01");

        // 7.使用会话对象创建主题订阅者
        TopicSubscriber subscriber = session.createDurableSubscriber(topic, "subscriber-1");

        // 8.通过监听器的方式来接收消息
        subscriber.setMessageListener(new MessageListener() {

            @Override
            public void onMessage(Message message) {
                try {
                    if (message instanceof TextMessage) {
                        System.out.println("已接收消息：" + ((TextMessage) message).getText());
                    }
                } catch (JMSException e) {
                    e.printStackTrace();
                }
            }
        });
    }
}
```

启动消费者，打开控制台页面，点击 "Subscribers"，可以看到以下页面。

![image](https://miansen.wang/assets/20200512112141.png)

在Active Durable Topic Subscribers里可以看到有一个消费者的信息，说明订阅成功了。

此时，我们将这个消费者线程关闭，再来看看，可以看到这个消费者跑到了Offline Durable Topic Subscribers里面，如下图所示。表明这个消费者离线了，但是依旧保持订阅关系。

![image](https://miansen.wang/assets/20200512112431.png)

下面做一个试验，此时消费者已经离线，我们先启动生产者，后启动消费者，此时消费者还能接收到刚才生产者发送的消息吗？

答案是可以的。

此时再刷新控制台，可以看到刚才在Offline的订阅者跑到了Active里面。

上述内容可以联系微信公众号理解。我们可以想象一个场景，我订阅了某公众号，后来我手机没电关机了，当我意识到后，充电再开机，我依旧能收到公众号推送给我的消息，这里的持久化，可以认为，我和公众号的联系被持久化了，所以我上线后可以及时收到消息。