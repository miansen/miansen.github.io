---
layout: post
title: 3.学习 Kafka - Java 客户端
date: 2020-05-21
categories: Kafka
tags: Kafka
author: 龙德
---

* content
{:toc}

引入客户端依赖：

```xml
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>2.5.0</version>
</dependency>
```

## 生产者

### 阻塞的生产者

生产者代码如下：

```java
public class CustomProducer {

    public static void main(String[] args) throws InterruptedException, ExecutionException {
        // 1.配置生产者的属性，各个属性的作用可以在这里看：http://kafka.apachecn.org/documentation.html#producerconfigs
        Properties props = new Properties();
        props.put("bootstrap.servers", "192.168.197.6:9092");
        props.put("acks", "all");
        props.put("retries", 0);
        props.put("batch.size", 16384);
        props.put("linger.ms", 1);
        props.put("buffer.memory", 33554432);
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        // 2.实例化一个生产者
        KafkaProducer<String, String> producer = new KafkaProducer<>(props);
        for (int i = 0; i < 100; i++) {
            // 3.实例化一条消息，第一个参数是主题的名称，第二个参数是消息的内容
            ProducerRecord<String,String> record = new ProducerRecord<>("test01", Integer.toString(i));
            // 4.发送消息
            Future<RecordMetadata> future = producer.send(record);
            // 5.获取发送的结果，注意这个方法会阻塞当前线程
            RecordMetadata metadata = future.get();
            System.out.println("topic: " + metadata.topic() + ", partition: " + metadata.partition() + ", offset: " + metadata.offset());
        }
        // 6.关闭资源
        producer.close();
    }
}
```

### 带有回调的生产者

前面说过，`future.get()` 方法会阻塞当前线程，这样会影响到后面代码的执行，好在 Kafka 客户端提供了回调的方法。

代码如下：

```java
public class CallbackProducer {

    public static void main(String[] args) throws InterruptedException, ExecutionException {
        // 1.配置生产者的属性，各个属性的作用可以在这里看：http://kafka.apachecn.org/documentation.html#producerconfigs
        Properties props = new Properties();
        props.put("bootstrap.servers", "192.168.197.6:9092");
        props.put("acks", "all");
        props.put("retries", 0);
        props.put("batch.size", 16384);
        props.put("linger.ms", 1);
        props.put("buffer.memory", 33554432);
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        // 2.实例化一个生产者
        KafkaProducer<String, String> producer = new KafkaProducer<>(props);
        for (int i = 0; i < 100; i++) {
            // 3.实例化一条消息，第一个参数是主题的名称，第二个参数是消息的内容
            ProducerRecord<String,String> record = new ProducerRecord<>("test01", Integer.toString(i));
            // 4.发送消息
            producer.send(record, new Callback() {
                // 5.在回调函数里处理结果，这样就不会阻塞了
                @Override
                public void onCompletion(RecordMetadata metadata, Exception exception) {
                    System.out.println("topic: " + metadata.topic() + ", partition: " + metadata.partition() + ", offset: " + metadata.offset());
                }
            });
        }
        // 6.关闭资源
        producer.close();
    }
}
```

### 消息发送到哪个分区？

![image](https://miansen.wang/assets/20200521165838.png)

如上图所示，ProducerRecord 类的构造函数有很多重载，根据构造函数来决定消息的分区，规则如下：

- 如果不指定分区和 key，那么消息会轮询负载均衡到每个分区上。

```java
// 轮询负载均衡到每个分区上
ProducerRecord<String,String> record = new ProducerRecord<>("test01", Integer.toString(i));
```

- 如果不指定分区，但是指定了 key，那么会将 key hash，然后取模，根据模数决定分区。

```java
// 消息发送到哪个分区取决于 key
ProducerRecord<String,String> record = new ProducerRecord<>("test01", Integer.toString(i), Integer.toString(i));
```

- 如果指定分区，那么直接发送到指定的分区。

```java
// 指定消息发送到第 2 个分区
ProducerRecord<String,String> record = new ProducerRecord<>("test01", 2, Integer.toString(i), Integer.toString(i));
```

## 消费者

### 自动提交偏移量

```java
public class AutoCommitConsumer {

    public static void main(String[] args) {
        // 1.配置消费者的属性，各个属性的作用可以在这里看：http://kafka.apachecn.org/documentation.html#consumerconfigs
        Properties props = new Properties();
        props.put("bootstrap.servers", "192.168.197.6:9092");
        props.put("group.id", "my-consumer-group-01");
        props.put("enable.auto.commit", "true");
        props.put("auto.commit.interval.ms", "1000");
        props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        // 2.实例化一个消费者
        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
        // 3.订阅主题（可以订阅多个）
        consumer.subscribe(Arrays.asList("test01"));
        // 4.循环消费消息
        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
            for (ConsumerRecord<String, String> record : records) {
                System.out.println(String.format("topic: %s, partition: %s, offset: %s, key: %s, value: %s",
                        record.topic(), record.partition(), record.offset(), record.key(), record.value()));
            }
        }
    }
}
```

通过配置 enable.auto.commit，表明这是个自动提交偏移量，偏移量由 auto.commit.interval.ms 控制自动提交的频率。

通过配置 bootstrap.servers 指定集群的一个或多个 broker。不用指定全部的 broker，它将自动发现集群中的其余的 borker（最好指定多个，万一有服务器故障）。

在这个例子中，客户端订阅了主题 test01。消费者组叫 my-consumer-group-01。

broker 通过心跳机制自动检测 my-consumer-group-01 组中失败的消费者实例，消费者实例会自动 ping 集群，告诉集群它还活着。只要消费者实例能够做到这一点，它就被认为是活着的，并保留分配给它分区的权利，如果它停止心跳的时间超过 session.timeout.ms 设置的值，那么就会认为是故障的，它的分区将被分配到别的消费者实例。

通过配置 deserializer 如何把 byte 转成 object 类型，例子中，通过指定 string 解析器，我们告诉获取到的消息的 key 和 value 只是简单个 string 类型。

### 手动提交偏移量

通过设置 props.put("enable.auto.commit", "false"); 改为手动提交偏移量。

然后在消费消息后手动提交。

```java
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
    for (ConsumerRecord<String, String> record : records) {
        System.out.println(String.format("topic: %s, partition: %s, offset: %s, key: %s, value: %s",
                record.topic(), record.partition(), record.offset(), record.key(), record.value()));
    }
    consumer.commitSync();
}
```

为什么需要手动提交？

可以想象一个场景：读取消息后，将它们批量插入到数据库中。如果我们设置 offset 自动提交，消费将被认为是已消费的。这样会出现问题，因为消息被插入到数据库之前可能会失败，这样我们就会丢失部分的数据。

```java
List<ConsumerRecord<String, String>> buffer = new ArrayList<>();
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
    for (ConsumerRecord<String, String> record : records) {
        buffer.add(record);
    }
    if (buffer.size() >= 100) {
        try {
            // 插入数据库
            insertIntoDb(buffer);
        } catch(Exception e) {
            // 如果插入数据库失败
            // TODO
        }
        // 插入数据库成功再手动提交
        consumer.commitSync();
        buffer.clear();
    }
}
```

### 订阅指定的分区

在前面的例子中，我们订阅感兴趣的 topic，让 Kafka 提供给我们平分后的分区。但是，在有些情况下，你可能需要自己来控制分配指定分区。

要使用此模式，你只需调用 assign(Collection<TopicPartition> partitions) 方法消费指定的分区即可：

```java
// 订阅分区
TopicPartition partition0 = new TopicPartition("test01", 0);
TopicPartition partition1 = new TopicPartition("test01", 1);
consumer.assign(Arrays.asList(partition0, partition1));
```

注意订阅主题和订阅分区是互斥的。

### 控制消费的位置

大多数情况下，消费者只是简单的从头到尾的消费消息，周期性的提交 offset（自动或手动）。Kafka 也支持消费者去手动的控制消费的位置，可以消费之前的消息也可以跳过最近的消息。

要使用此模式，你只需调用 
seek(TopicPartition partition, long offset) 方法消费指定的位置即可：

```java
TopicPartition partition0 = new TopicPartition("test01", 0);
consumer.seek(partition0, 784);
```

结果如下图：

![image](https://miansen.wang/assets/20200522181758.png)