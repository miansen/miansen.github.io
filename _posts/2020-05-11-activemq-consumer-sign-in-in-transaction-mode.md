---
layout: post
title: 12.学习ActiveMQ-事务模式下消费者签收
date: 2020-05-11
categories: 消息队列
tags: ActiveMQ
author: 龙德
---

* content
{:toc}

前面讲的签收是没有事务的，如果有事务，签收又是怎么样的呢？来看下面的实验。

将connection.createSession()的第一个参数改为true，表示消费者采用事务的方式来消费消息。

同时将第二个参数改为Session.CLIENT_ACKNOWLEDGE，表示手动签收消息。

![image](https://miansen.wang/assets/20200511152526.png)

## 第一种情况

写 session.commit() 方法，不写 TextMessage.acknowledge() 方法。

![image](https://miansen.wang/assets/20200511153238.png)

依次启动生产者和消费者，这时候，消费者可以拿到消息。

把消费者关了再启动，却不会出现重复消费的情况，这是为什么呢？

我们可以理解为执行了 session.commit() 操作，也就告知了 MQ，消费者已经完成了签收，这里的 CLIENT_ACKNOWLEDGE 也就相当于 AUTO_ACKNOWLEDGE 了。

## 第二种情况

不写 session.commit() 方法，写 TextMessage.acknowledge() 方法。

![image](https://miansen.wang/assets/20200511153542.png)

依次启动生产者和消费者，这时候，消费者可以拿到消息。

把消费者关了再启动，发现消费者重复消费了消息。

## 结论

通过上面两种情况，我们可以得出结论：事务的作用 > 签收的作用。

- 在事务模式下，当一个事务被成功提交，则消息被自动签收，如果事务回滚，消息会被再次传送。

- 在非事务模式下，消息何时被确认取决于创建会话的签收模式。如果是 AUTO_ACKNOWLEDGE 模式，则自动签收。如果是 CLIENT_ACKNOWLEDGE 模式，则调用 TextMessage.acknowledge() 方法签收。