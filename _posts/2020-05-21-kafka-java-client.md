---
layout: post
title: 3.学习Kafka-Java客户端
date: 2020-05-21
categories: Kafka
tags: Kafka
author: 龙德
---

* content
{:toc}

## 生产者

引入客户端依赖：

```xml
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>2.5.0</version>
</dependency>
```

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

## 带有回调的生产者

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