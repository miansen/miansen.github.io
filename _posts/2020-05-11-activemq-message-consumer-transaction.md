---
layout: post
title: 十、学习ActiveMQ-消息消费者的事务
date: 2020-05-11
categories: ActiveMQ
tags: ActiveMQ
author: 龙德
---

* content
{:toc}

事务主要是针对生产者而言的，但是对于消费者，也有事务。

将connection.createSession()的第一个参数改为true，表示消费者采用事务的方式来消费消息。

![image](https://miansen.wang/assets/20200511135735.png)

通过前面的介绍，我们知道，如果开启了事务，就需要手动执行session.commit()来提交事务，假设这时候，我们不提交事务，看看会出现什么情况。

我们先启动生产者。

可以看到生产者生产了一条消息。

![image](https://miansen.wang/assets/20200511140128.png)

![image](https://miansen.wang/assets/20200511140157.png)

接着启动消费者。

这时候消费者是可以消费到消息的。

![image](https://miansen.wang/assets/20200511140526.png)

但是注意看控制台，这时候队列中还存在1条消息等待消息，有0条消息已消费。

![image](https://miansen.wang/assets/20200511140634.png)

这时候我们如果再启动一个消费者，发现又做了一次消费，也就是重复消费了。

也就是说在事务模式下，消费者虽然消费到了消息，但是它没有提交事务，也就没有通知到MQ，所以MQ并不知道消息是否被消费了，所以就会出现重复消费的情况。

所以消费者如果开启了事务，那么也要提交事务。否则MQ并不知道消息是否被消费了，最后就会出现重复消费的情况。