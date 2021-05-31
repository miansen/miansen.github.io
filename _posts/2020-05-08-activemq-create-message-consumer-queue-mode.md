---
layout: post
title: 4.学习ActiveMQ-消息消费者（队列模式）
date: 2020-05-08
categories: 消息队列
tags: ActiveMQ
author: 龙德
---

* content
{:toc}

消费者消费消息有两种方式，第一种方式是使用 receive() 方法，这种方式是同步阻塞的，会阻塞等待生产者生产消息。第二种方式是使用 MessageListener，这种方式是异步非阻塞的，当有消息到达的时候，会调用它的 onMessage() 方法。

## 第一种方式：receive()

```java
public class QueueConsumer {

    public static void main(String[] args) {
        
        Connection connection = null;
        Session session = null;
        MessageConsumer consumer = null;
        
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

            // 5.使用会话对象创建目标对象，这里我们创建的是 queue，也就是点对点的模式。与生产者保持一致，并且名称也要一致。
            Destination destination = session.createQueue("queue-test-01");

            // 6.使用会话对象创建消费者对象
            consumer = session.createConsumer(destination);

            // 7.使用 receive() 方法消费消息，当没有消息的时候会阻塞在这，等待提供者提供消息
            TextMessage msg = (TextMessage) consumer.receive();
            System.out.println("已接收消息：" + msg.getText());
        } catch (JMSException e) {
            e.printStackTrace();
        } finally {
            // 8.关闭资源
            if (consumer != null) {
                try {
                    consumer.close();
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

运行消费者，查看控制台，可以看到消费了 1 条消息，有 0 条消息等待消费。

![image](https://miansen.wang/assets/20200507160537.png)

由于队列中已经存在一条消息了，所以看不到阻塞的效果。我们重新启动生产者，就可以看到阻塞在了 receive() 处。

![image](https://miansen.wang/assets/20200509172606.png)

此时如果没有消息进入队列中，消费者就会一直阻塞着。

如何做到一直接收消息呢，总不能接收到一条消息就关了吧。我们可以使用 while() 循环一直接收消息

```java
while (true) {
    TextMessage msg = (TextMessage) consumer.receive();
    System.out.println("已接收消息：" + msg.getText());
}
```

但这种方式毕竟会阻塞当前线程，如果使用这种方式，那么后面的代码也就别执行了。所以我们可以另起一条线程来解决阻塞的问题。

```java
public class QueueConsumer {

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

        // 7.使用 receive() 方法消费消息，当没有消息的时候会阻塞，我们可以开一个线程解决阻塞的问题
        new Thread("My-ActiveMQ-Thread") {

            @Override
            public void run() {

                try {
                    while (true) {
                        TextMessage msg = (TextMessage) consumer.receive();
                        System.out.println("已接收消息：" + msg.getText());
                    }
                } catch (JMSException e) {
                    e.printStackTrace();
                }

            }
        }.start();
    }
}
```

## 第二种方式：MessageListener

```java
public class QueueConsumer {

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

        // 7.通过监听器的方式来接收消息
        consumer.setMessageListener(new MessageListener() {

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