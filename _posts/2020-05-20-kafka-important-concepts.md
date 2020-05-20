---
layout: post
title: 2.学习Kafka-重要概念
date: 2020-05-20
categories: Kafka
tags: Kafka
author: 龙德
---

* content
{:toc}

由于搭建集群的时候，3 台虚拟机都在前台开启了 Kafka，占用了窗口，所以我这里再开启两个窗口，分别用来操作 Kafka 和 Zookeeper。

![image](https://miansen.wang/assets/20200519173218.png)

下面的操作都在这两个窗口上进行。

## 主题（Topic）

如果把 Kafka 比作 MySQL 的话，那么主题就相当于数据库，所以想用 Kafka，就必须先创建主题，就好比我们想用 MySQL，就必须先创建数据库一样。

使用这个命令创建主题：

```shell
/usr/local/kafka_2.13-2.5.0/bin/kafka-topics.sh --create --zookeeper 192.168.197.6:2181 --replication-factor 3 --partitions 3 --topic test01
```

关于这条命令的解释：

--zookeeper：指定 zookeeper 集群的 ip 地址和端口

--topic：创建名为 test01 的主题

--partitions：test01 这个主题分配 3 个分区

--replication-factor：每个分区分配 3 个副本，也就是每个分区备份 3 份

主题在 Zookeeper 中是如何体现的呢，我们进入 Zookeeper 看一下。

使用命令 `ls /brokers/topics`，可以看到刚才创建的 test01 主题在 Zookeeper 中已经创建了。

![image](https://miansen.wang/assets/20200520111038.png)

使用命令 `/brokers/topics/test01/partitions`，可以看到分区也创建了。

![image](https://miansen.wang/assets/20200520111218.png)

## 分区（partition）

主题只是一个逻辑的概念，在物理上并不存在，消息也不是存在主题里的，而是存在分区里的。分区是 Kafka 消息存储的最小单元，在物理上对应着一个个真实的文件。

我们上面新建了一个 test01 主题，分配了 3 个分区，这 3 个分区其实就是一个个的文件夹。我们可以进入 log.dirs 目录

```shell
cd /usr/local/kafka_2.13-2.5.0/logs/
```

![image](https://miansen.wang/assets/20200520134056.png)

可以看到创建了 3 个文件夹，分别对应 3 个分区，命名规则是 `topic名称-N`，N 就是分区的个数。后面生产者发送给 test01 主题的数据都会存在这 3 个文件夹里。

进入 test01 主题的第一个分区看一下

![image](https://miansen.wang/assets/20200520134919.png)

可以看到每个分区文件夹，都包含以下 4 类文件：

- xxxxxxxxx.index：索引文件
- xxxxxxxxx.log：数据文件
- xxxxxxxxx.timeindex：时间戳索引文件
- leader-epoch-checkpoint：保存了每一任 leader 开始写入消息时的 offset，会定时更新，当
follower 被选为 leader 时会根据这个确定哪些消息可用

我们重点看 .index .log .timeindex 这 3 个文件。

这 3 个文件都是成对出现的，每一对都叫做 segment。也就是说分区中的数据是分段存储，一个分区（partition）被切割成多个相同大小的段（segment）（这个是由 log.segment.bytes 决定，控制每个 segment 的大小）。

segment 文件命名规则：partition 全局的第一个 segment 从 0 开始，后续每个 segment 文件名为上一个 segment 文件最后一条消息的 offset 值。数值最大为 64 位 long 大小，19 位数字字符长度，没有数字用 0 填充。

如下图所示：

![image](https://miansen.wang/assets/20200520141558.png)

通过上面的内容，我们可以这样总结：

在 Kafka 文件存储中，同一个 topic 下有多个不同的 partition，每个 partiton 为一个目录，目录的命名规则为 `topic名称+有序序号`。第一个序号从 0 开始计，最大的序号为 partition 数量减 1。partition 是实际物理上的概念，而 topic 是逻辑上的概念，topic 更像是一个消息的类别。由于生产者生产的消息会不断追加到 log 文件的末尾，为防止 log 文件过大导致数据定位效率低下，Kafka 将 partition 细分为 segment，一个 partition 物理上由多个 segment 组成。每个 segment 由 3 个文件组成，分别是 index 文件、log 文件和 timeindex 文件，这 3 个文件的命名规则是：partition 全局的第一个 segment 从 0 开始，后续每个 segment 文件名为上一个 segment 文件最后一条消息的 offset 值。

## 偏移量（offset）

分区中的每一条记录都会分配一个 id 号来表示顺序，我们称之为 offset，offset 用来唯一的标识分区中每一条记录。

每一个消费者中唯一保存的元数据是 offset（偏移量）即消费在 log 中的位置。偏移量由消费者所控制：通常在读取记录后，消费者会以线性的方式增加偏移量，但是实际上，由于这个位置由消费者控制，所以消费者可以采用任何顺序来消费记录。例如，一个消费者可以重置到一个旧的偏移量，从而重新处理过去的数据。也可以跳过最近的记录，从 "现在" 开始消费。

特别说明的是，老版本的消费者是依赖 Zookeeper 的，当启动一个消费者时向 Zookeeper 注册。新版本消费者去掉了对 Zookeeper 的依赖，而是由消费组协调器统一管理，已消费的消息偏移量提交后会保存到名为 "__consumer_offsets" 文件夹中。

![image](https://miansen.wang/assets/20200520175330.png)

Kafka 的 offset 是分区内有序的，但是在不同分区中是无序的，Kafka 不保证数据的全局有序。

关于 offset，可以通过以下几个问题来理解：

1.为啥 offset 在分区内是有序的？

因为分区的结构是队列，先进先出，所以保证了有序。

2.为啥在不同分区中是无序的？

因为消息写入哪一个分区不是固定的，有可能第一条消息写入第一个分区，第二条消息写入第二个区分。。。

3.如果有业务场景需要保证数据的有序呢？该如何做？

可以这样做：创建主题的时候只分配一个分区，这样就能保证数据的顺序性了。

但是只分配一个分区，又跟 Kafka 的分布式冲突，所以还可以这样做：

创建主题的时候分配多个分区，读取消息时，在代码层面处理消息的顺序性。

## broker

一个 Kafka 实例就是 broker，一个 broker 有多个 topic，一个 topic 有多个 partition，一个 partition 有多个 segment。

![image](https://miansen.wang/assets/20200520160017.png)

## 生产者（producer）

Kafka 自带一个命令行客户端，它从文件或标准输入中获取输入，并将其作为 message（消息）发送到 Kafka 集群。默认情况下，每行将作为单独的 message 发送。

运行 producer，然后在控制台输入一些消息以发送到服务器。

```shell
[root@localhost kafka_2.13-2.5.0]# /usr/local/kafka_2.13-2.5.0/bin/kafka-console-producer.sh --broker-list 192.168.197.6:9092 --topic test01
>This is a message
>This is another message
>
```

## 消费者（consumer）

Kafka 还有一个命令行 consumer（消费者），将消息转储到标准输出。

```shell
/usr/local/kafka_2.13-2.5.0/bin/kafka-console-consumer.sh --bootstrap-server 192.168.197.12:9092 --topic test01 --from-beginning
This is a message
This is another message
```

## 消费组（consumer-groups）

消费者使用一个 "消费组" 名称来进行标识，发布到 topic 中的每条记录被分配给订阅消费组中的一个消费者实例。消费者实例可以分布在多个进程中或者多个机器上。

如果所有的消费者实例在同一消费组中，消息记录会负载平衡到每一个消费者实例。

如果所有的消费者实例在不同的消费组中，每条消息记录会广播到所有的消费者进程。

![image](https://miansen.wang/assets/kafka-consumer-groups.png)

如图，这个 Kafka 集群有两台 server 的，四个分区(p0-p3)和两个消费者组。消费组A有两个消费者，消费组B有四个消费者。

通常情况下，每个 topic 都会有一些消费组，一个消费组对应一个"逻辑订阅者"。一个消费组由许多消费者实例组成，便于扩展和容错。这就是发布和订阅的概念，只不过订阅者是一组消费者而不是单个的进程。

在Kafka中实现消费的方式是将日志中的分区划分到每一个消费者实例上，以便在任何时间，每个实例都是分区唯一的消费者。维护消费组中的消费关系由Kafka协议动态处理。如果新的实例加入组，他们将从组中其他成员处接管一些 partition 分区;如果一个实例消失，拥有的分区将被分发到剩余的实例。

Kafka 只保证分区内的记录是有序的，而不保证主题中不同分区的顺序。每个 partition 分区按照key值排序足以满足大多数应用程序的需求。但如果你需要总记录在所有记录的上面，可使用仅有一个分区的主题来实现，这意味着每个消费者组只有一个消费者进程。

创建消费者时如果不指定消费组，那么就会默认分配一个，可以通过这个命令查看有哪些消费组。

```shell
/usr/local/kafka_2.13-2.5.0/bin/kafka-consumer-groups.sh --bootstrap-server 192.168.197.6:9092 --list
```

![image](https://miansen.wang/assets/20200520180552.png)

如图，因为我启动了 4 个消费者，所以有 4 个消费组。由于这 4 个消费者都属于不同的消费组，所以他们都能收到 test01 主题的所有消息。