---
layout: post
title: 三、学习ActiveMQ-Session
date: 2020-05-08
categories: ActiveMQ
tags: ActiveMQ
author: 龙德
---

* content
{:toc}

前面提到创建 Session 时有两个参数，第一个参数的作用是开启事务，false 不开启事务，true 就是开启事务，如果开启事务要在关闭连接之前 commit() 一下，否则消息不会进入到 AMQ。

![image](https://miansen.wang/assets/20200507165915.png)

我们开启改成 true，开启事务

![image](https://miansen.wang/assets/20200507170202.png)

然后运行生产者，查看控制台，可以看到消息并没有进入到 AMQ。

![image](https://miansen.wang/assets/20200507170408.png)

我们在关闭连接之前 commit() 一下

![image](https://miansen.wang/assets/20200507170519.png)

这样消息才能进入到 AMQ。

![image](https://miansen.wang/assets/20200507170929.png)