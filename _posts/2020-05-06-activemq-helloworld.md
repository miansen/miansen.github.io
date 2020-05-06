---
layout: post
title: 二、学习ActiveMQ-Hello World
date: 2020-05-06
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

## 创建消息提供者

```java
// 1.创建连接工厂，需要传入ip和端口号，这里我们使用默认的
ActiveMQConnectionFactory connectionFactory = new ActiveMQConnectionFactory(
        ActiveMQConnectionFactory.DEFAULT_BROKER_BIND_URL);

// 2.使用连接工厂创建一个连接对象
Connection connection = connectionFactory.createConnection();

// 3.开启连接
connection.start();

// 4.使用连接对象创建会话对象。
// 这里有两个参数：第一个参数的作用是开启事务，false就是开启事务，
// true就是开启事务，如果开启事务要在关闭连接之前 commit() 一下，否则消息不会进入到 AMQ。
// 第二个参数表示消费模式，AUTO_ACKNOWLEDGE 表示自动消费，除此还有手动消费。
Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);

// 5.使用会话对象创建目标对象，这里我们创建的是 queue，也就是点对点的模式。除此之外还有一对多模式：createTopic()
Destination destination = session.createQueue("queue-test-01");

// 6.使用会话对象创建生产者对象
MessageProducer producer = session.createProducer(destination);

// 7.使用会话对象创建一个消息对象
TextMessage textMessage = session.createTextMessage();

// 8.设置消息内容
textMessage.setText("hello activemq");

// 9.发送消息
producer.send(textMessage);

// 10.关闭资源
producer.close();
session.close();
connection.close();
```

程序运行之后，打开控制台，点击 "Queues"，可以看到如下图：

<center>

![image](https://miansen.wang/assets/20200506185229.png)

</center>

> Messages Enqueued：表示生产了多少条消息，记做P
>
> Messages Dequeued：表示消费了多少条消息，记做C
> 
> Number Of Consumers：表示在该队列上还有多少消费者在等待接受消息
> 
> Number Of Pending Messages：表示还有多少条消息没有被消费，实际上是表示消息的积压程度，就是P-C

## 创建消息消费者

```java
// 1.创建连接工厂，需要传入ip和端口号，这里我们使用默认的
ActiveMQConnectionFactory connectionFactory = new ActiveMQConnectionFactory(
        ActiveMQConnectionFactory.DEFAULT_BROKER_BIND_URL);

// 2.使用连接工厂创建一个连接对象
Connection connection = connectionFactory.createConnection();

// 3.开启连接
connection.start();

// 4.使用连接对象创建会话对象。
// 这里有两个参数：第一个参数的作用是开启事务，false就是开启事务，
// true就是开启事务，如果开启事务要在关闭连接之前 commit() 一下，否则消息不会进入到 AMQ。
// 第二个参数表示消费模式，AUTO_ACKNOWLEDGE 表示自动消费，除此还有手动消费。
Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);

// 5.使用会话对象创建目标对象，这里我们创建的是 queue，也就是点对点的模式。除此之外还有一对多模式：createTopic()
Destination destination = session.createQueue("queue-test-01");

// 6.使用会话对象创建消费者对象
MessageConsumer consumer = session.createConsumer(destination);

// 7.receive() 方法当没有消息的时候会阻塞在这，等待提供者提供消息，后面介绍使用监听的方式来获取消息
TextMessage msg;
while (true) {
    msg = (TextMessage) consumer.receive();
    System.out.println("消费消息：" + msg.getText());
}
```

程序运行之后，消费者消费了两条消息。

<center>

![image](https://miansen.wang/assets/20200506190618.png)

</center>

我们继续生产一条消息

```java
textMessage.setText("hello activemq 01");
```

可以看到刚才生产了一条消息

<center>

![image](https://miansen.wang/assets/20200506190757.png)

</center>

接着消费者就消费了

<center>

![image](https://miansen.wang/assets/20200506190903.png)

</center>

生产者生产 100 条消息

<center>

![image](https://miansen.wang/assets/20200506191259.png)

</center>

消费者也消费了 100 条消息

<center>

![image](https://miansen.wang/assets/20200506191506.png)

</center>