---
layout: post
title: 一、学习ActiveMQ-关于JMS
date: 2020-05-05
categories: ActiveMQ
tags: ActiveMQ
author: 龙德
---

* content
{:toc}

## JMS的概念

百度百科的描述：

> JMS即Java消息服务（Java Message Service）应用程序接口，是一个Java平台中关于面向消息中间件（MOM）的API，用于在两个应用程序之间，或分布式系统中发送消息，进行异步通信。

看到 "接口" 两个字，下意识就想到了规范。没错，JMS 和 JDBC、JPA 一样，只是一个规范，也就是只给出了接口，具体的内容由各个厂商去实现。

![image](https://miansen.wang/assets/20200506143952.png)

## JMS的组成

![image](https://miansen.wang/assets/20200102060824826.png)

从上图可以看出，JMS 的组成有4个部分，它们分别是：

- JMS provider：实现 JMS 接口规范的消息中间件，也就是 MQ 服务器
- JMS producer：消息生产者，创建和发送JMS消息的客户端应用
- JMS consumer：消息消费者，接收和处理JMS消息的客户端应用
- JMS message：消息头、消息属性、消息体

## JMS的实现产品

MQ 产品有很多种，以下是最常见的 4 种：

![image](https://miansen.wang/assets/20200508181345.png)

我们接下来学习的 ActiveMQ 就是 apache 出品的一个非常流行的消息中间件。