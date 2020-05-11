---
layout: post
title: 十一、学习ActiveMQ-非事务模式下消费者签收
date: 2020-05-11
categories: ActiveMQ
tags: ActiveMQ
author: 龙德
---

* content
{:toc}

事务主要是偏向生产者，签收主要是偏向消费者。

之前我们说过，在创建 session 的时候，我们传了两个参数，第一个是事务，第二个是签收，在这篇博客中，我们来说说签收。

签收模式主要有以下4种：

- AUTO_ACKNOWLEDGE

自动签收消息。只要有消息就消费。

对于这种模式，可以理解为快递员自动给你签收了并放到了快递柜中。

- CLIENT_ACKNOWLEDGE

手动签收消息。需要在消费端调用一下 TextMessage.acknowledge() 方法才会签收。

对于这种模式，可以理解为见到快递员，开箱验货，再签收。

- DUPS_OK_ACKNOWLEDGE

表示 session 不必确保对传送消息的签收，它可能会引起消息的重复，但是降低了 session 开销，所以消费者需要能容忍重复的消息。在第二次重新传递消息的时候，消息头的 JmsDelivered 会被置为 true 标示当前消息已经传送过一次，消费者需要进行消息的重复处理控制
- SESSION_TRANSACTED

表示消息生产者提供了消息发送的事务处理方式。也就是指，消息生产者发送消息给 broker 后，broker 只是暂存该消息，只有当生产者给 broker 进行事务确认消息后，broker 才把消息加入到待发送队列中，换言之，如果消息发送者进行了事务回滚，消息会直接从 broker 中删除

第一种前面已经见过了，就不介绍了。后面那两种用得比较少，也不介绍了。我们来看看第二种。

将connection.createSession()的第二个参数改为Session.CLIENT_ACKNOWLEDGE，表示手动签收消息。

![image](https://miansen.wang/assets/20200511150618.png)

然后启动生产者生产1条消息。

接着启动消费者消费消息。

![image](https://miansen.wang/assets/20200511150928.png)

可以看到消费者消费了1条消息。

但是注意看控制台，这时候队列中还存在1条消息等待消息，有0条消息已消费。

![image](https://miansen.wang/assets/20200511151002.png)

这时候如果我们把消费者关了再启动，发现又做了一次消费，也就是重复消费了。

所以需要在消费端调用一下 TextMessage.acknowledge() 方法才会签收，否则会重复消费。

![image](https://miansen.wang/assets/20200511151731.png)