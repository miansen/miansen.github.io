---
layout: post
title: 模拟复现 RocketMQ 消费者重复消费消息的场景
date: 2021-05-31
categories: 消息队列
tags: RocketMQ
---

* content
{:toc}

这篇文章模拟复现 RocketMQ 消费者重复消费消息的场景，一共有 3 种场景，分别是：

- 网络闪断
- 负载均衡
- 应用重启

## 第一种场景：网络闪断

首先我用生产者生产了一条消息，如下图所示：

![image.png](https://i.loli.net/2021/05/30/eGELlBuz8mN6PnO.png)

然后消费者消费消息，但是我打了一个断点，不让消费者提交 commit，模拟网络闪断的场景。这样 broker 就收不到这条消息的消费结果。

![image.png](https://i.loli.net/2021/05/30/Z1OUy6XpQ2Rbgcs.png)

![image.png](https://i.loli.net/2021/05/30/wFSIviGpKqLlnEr.png)

如上图所示，第一条消息的处理线程是 ConsumerMessageThread_2，投递时间是 21:41:16，messageId 是 8197be43cfca1c937a7b53edf7f1265e

由于我打了断点，broker 一直收不到这条消息的消费结果，所以大概过了 1809661 毫秒，broker 认为这条消息没有消费成功，所以又在 22:06:24 重新投递了一次。所以上图显示投递次数为 2。

![image.png](https://i.loli.net/2021/05/30/QMDEjs13TqSg5Lr.png)

再如上图所示，由于 broker 又投递了一次消息，所以消费者又消费了同一条消息，第二条消息的处理线程是 ConsumerMessageThread_3，投递时间是 22:06:24，messageId 也是 8197be43cfca1c937a7b53edf7f1265e

这样就大致模拟出了：**在消息消费的场景下，消息已投递到消费者并完成业务处理，当客户端给服务端反馈应答的时候网络闪断。为了保证消息至少被消费一次，消息队列 RocketMQ 的服务端将在网络恢复后再次尝试投递之前已被处理过的消息，消费者后续会收到两条内容相同并且 Message ID 也相同的消息。**

在我写这个文章的过程中，消费者又消费了一次。所以到目前为止，消费者一共消费了同一条消息 3 次。

![image.png](https://i.loli.net/2021/05/30/QNj2MynFC17oYTp.png)

![image.png](https://i.loli.net/2021/05/30/WsdAXFc7DpokvE6.png)

![image.png](https://i.loli.net/2021/05/30/ZXhNCwH31afSTYK.png)

我放开最后一条消息的断点，让这条消息消费完并提交 commit，这条消息的状态就变成了“消费成功”。

![image.png](https://i.loli.net/2021/05/30/xLsmh1nVcKRX2rZ.png)

由于有一个消费者消费成功并提交了 commit，broker 收到了 commit，就会认为这条消息已经被这个消费者组里的某个消费者消费到了，后续就不会再投递这条消息给这个消费者组里的消费者了。

接着我放开剩下两个线程的断点，可以看到所有的消息的状态都变成了 “消费成功”。

![image.png](https://i.loli.net/2021/05/30/4YwSmQBGfhabJCD.png)

## 第二种场景：负载均衡

首先我用生产者生产一条消息，然后消费者A（192.168.3.243）在今天 13:38:32 时消费了消息，并向 broker 提交了 commit。如下图所示：

![image.png](https://i.loli.net/2021/05/31/xLR451foFNhm9cp.png)

下面是应用A打印的消费日志，这个日志不重要，主要想说明的是在 13:38:54 时，消费者确实消费到了消息。

```java
2021-05-31 13:38:54.921|INFO|20523a3ff8dc08caa23b0eb7bd3140f0-|ConsumeMessageThread_1|cn.tanzhou.selfsales.manager.component.FlowAddFriendComponent|147|addFriendHandle|invoke method addFriendHandle. params: {"createUid":198937991,"flowAddFriendId":3,"randomNumber":97}
```

本来这条消息已经被成功消费了，按道理来说同一个消费者组下的其他消费者不会再消费这条消息了。但是当我再启动一个消费者B（169.254.189.17）时，由于这个消费者B跟消费者A同处于一个消费者组下，相当于消费者扩容了，所以会触发 Rebalance，那么此时消费者B可能会消费到重复消息。

如下图所示，消费者B重复消费了消息。

![image.png](https://i.loli.net/2021/05/31/2Ujg7VGwHYoQhcm.png)

这是应用B打印的消费日志：

```java
2021-05-31 13:52:38.384|INFO|67cf0a00f7fe653f2c3fa37dcd32f5cb-|ConsumeMessageThread_1|cn.tanzhou.selfsales.manager.event.consumer.FlowAddFriendConsumer|43|process|FlowAddFriendConsumer result {"createUid":198937991,"flowAddFriendId":4,"randomNumber":97}
```

这样就大致模拟出了：**当消息队列 RocketMQ 的 Broker 或客户端重启、扩容或缩容时，会触发 Rebalance，此时消费者可能会收到重复消息。**

## 第三种场景：应用重启

假设我们的应用在线上部署了两台节点，分别是 192.168.3.243 和 10.0.65.145。我们有新功能上线需要发版，当我们发版时，第一台节点正好消费到了消息，但是还没有消费完，还在执行业务逻辑。如果这时候第一台应用重启了，broker 收不到消费者A的应答信息，就会认为这条消息没有被成功消费，那么就会将这条消息重新投递，这时候会被第二台应用的消息者消费。

![image.png](https://i.loli.net/2021/05/31/GFALCxNPt8EbeYs.png)

如上图所示，这条消息在今天 14:53:52 时被应用A的消费者消费了，但是应用A刚好在消费的过程中重启了，没有提交应答信息，导致 broker 认为这条没有被成功消费，所以又在 14:54:02 时重新投递，被应用B的消费者消费到了。

## 总结

以上内容模拟复现 RocketMQ 消费者重复消费消息的场景，用文字和图片记录下来，以便在脑海中有一个清晰的认识。以后出现消费者对某条消息重复消费的情况时，我们可以知道消费者为什么会重复消息，以及知道如何对消息做幂等处理。

如何处理消费者重复消费消息，请参考这篇文章：[消费幂等](https://help.aliyun.com/document_detail/44397.html)