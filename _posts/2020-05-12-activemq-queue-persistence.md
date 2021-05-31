---
layout: post
title: 13.学习ActiveMQ-队列的持久化
date: 2020-05-12
categories: 消息队列
tags: ActiveMQ
author: 龙德
---

* content
{:toc}

## 持久化

持久化队列只需要设置 producer.setDeliveryMode(DeliveryMode.PERSISTENT) 就可以了。默认是持久化的。

我们可以做个试验，使用 producer.setDeliveryMode(DeliveryMode.PERSISTENT) 设置为持久化模式，先用生产者产生3条消息发送给MQ，然后手动关闭MQ，再启动MQ，然后用consumer去消费，可以发现消费者依然可以拿到消息，也就是消息在宕机之前已经被存储了下来，后面再启动的时候，会读取到存储的数据。

## 非持久化

非持久化队列也很简单，只需要设置 producer.setDeliveryMode(DeliveryMode.NON_PERSISTENT) 就可以了。

我们再做个试验，使用 producer.setDeliveryMode(DeliveryMode.NON_PERSISTENT) 设置为非持久化模式，先用生产者产生3条消息发送给MQ，然后手动关闭MQ，再启动MQ，然后用consumer去消费，可以发现消费者什么都拿不到，也就是消息已经丢了，我们在MQ的控制台页面看到，MQ里的消息数量是0。