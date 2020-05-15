---
layout: post
title: 15.学习ActiveMQ-持久化数据到MySQL
date: 2020-05-12
categories: ActiveMQ
tags: ActiveMQ
author: 龙德
---

* content
{:toc}

ActiveMQ 的持久化可以这样理解：生产者生产消息之后，消息首先会到达 broker，broker 就相当于一个调度中心。broker 首先将消息存储在本地的数据文件（或者是远程数据库），再将消息发送给消费者。如果消费者接收到了消息，则将消息从存储中删除，否则继续尝试重发。

ActiveMQ 启动后，首先检查存储位置，如果有未发送成功的消息，需要先把消息发送出去。

ActiveMQ 默认的持久化数据库是 KahaDB，我们也可以改成 MySQL。

第一步，首先将 mysql-connector-java 的jar 扔到 ActiveMQ的lib目录下。

![image](https://miansen.wang/assets/20200512134504.png)

然后修改 conf 目录下的 activmeq.xml 中的 persistenceAdapter 结点为 JDBC。

```xml
<persistenceAdapter>
    <!-- <kahaDB directory="${activemq.data}/kahadb"/> -->
    <jdbcPersistenceAdapter dataSource="#mysql-datasource"/>
</persistenceAdapter>
```

再添加 mysql-datasource，这样上面的配置文件才能拿到 DataSource。将下面这段内容放在 broker 结点和 import 结点之间。

```xml
<bean id="mysql-datasource" class="org.apache.commons.dbcp2.BasicDataSource" destroy-method="close"> 
    <property name="driverClassName" value="com.mysql.jdbc.Driver"/> 
    <property name="url" value="jdbc:mysql://localhost:3306/activemq?useUnicode=true&amp;characterEncoding=UTF-8"/> 
    <property name="username" value="root"/> 
    <property name="password" value="123"/> 
    <property name="poolPreparedStatements" value="true"/> 
</bean>
```

上面的用户名和密码要改成你自己的，还有要注意 url 参数，这里的`&`要改成`&amp;`，也就是&的转义，否则启动会报错。

启动 ActiveMQ，打开 activemq 库，就会发现自动生成了 3 张表。

![image](https://miansen.wang/assets/20200512140556.png)

每张表的作用如下：

activemq_msgs（消息表）：

![image](https://miansen.wang/assets/20200512140723.png)

activemq_acks（订阅关系表，针对于持久化的Topic，订阅者和服务器的订阅关系会保存在这个表中）：

![image](https://miansen.wang/assets/20200512140822.png)


activemq_lock：

在集群环境中才有用，只有一个 Broker 可以获取消息，称为 Master Broker，其他的只能作为备份等待 Master Broker 不可用，才可能成为下一个 Master Broker。这个表用来记录哪个 Broker 是当前的 Master Broker。

### 测试队列的数据

运行队列生产者生产 3 条数据，然后查看 activemq_msgs 表，可以看到这 3 条消息已经存进 activemq_msgs 表了。

![image](https://miansen.wang/assets/20200512141554.png)

然后启动队列消费者消费消息，在刷新 activemq_msgs 表，可以看到数据没了，说明消息的消费操作也同步更新了数据库。

![image](https://miansen.wang/assets/20200512141841.png)

### 测试主题的数据

启动消费者 DurableTopicConsumer 类，查看数据库的 activemq_acks 表，可以发现订阅关系已经持久化到数据库里了。

![image](https://miansen.wang/assets/20200512143406.png)

启动生产者 DurableTopicProducer 类，这时候消息已经被消费了。但是再查看数据库的 activemq_msgs 表，里面的数据依然存在。

![image](https://miansen.wang/assets/20200512143902.png)

上述内容可以联系微信公众号来理解，微信公众号发布过的文章，不能因为订阅者已经收到了或者是服务器关闭了，就把发布过的文章和订阅者清空，这是不合理的，从这个角度来理解就能明白了。

<span id="issueId" style="display: none;">3</span>