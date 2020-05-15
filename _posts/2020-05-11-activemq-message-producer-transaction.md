---
layout: post
title: 9.学习ActiveMQ-消息生产者的事务
date: 2020-05-11
categories: ActiveMQ
tags: ActiveMQ
author: 龙德
---

* content
{:toc}

之前我们说过，在创建 session 的时候，我们传了两个参数，第一个是事务，第二个是签收，在这篇博客中，我们来说说事务。

connection.createSession()方法的第一个参数就是事务，它的值可以是true或false，代表session的提交是事务提交还是非事务提交。

当事务的值是false时，只要执行了producer.send()方法，消息就到了队列中，也就是自动提交了。

当事务的值是true时，在执行完roducer.send()方法后，在session关闭之前需要多加一个session.commit()方法提交事务。

第一个参数传入 true，这样就开启事务了。

![image](https://miansen.wang/assets/20200511134202.png)

session关闭之前提交事务，否则消息不会进入队列中。

![image](https://miansen.wang/assets/20200511134507.png)

事务的提交，用于实际复杂的业务场景，可能有多个消息需要入队列，假设有一条入队列报错了，我们希望这一批次的都要回滚，这时候就要用到session.rollback()方法了，可以将session.rollback()方法放在catch语句块中来执行。

![image](https://miansen.wang/assets/20200511135029.png)