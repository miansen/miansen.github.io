---
layout: post
title: 5.学习 Kafka - 在 Spring Boot 中使用 Kafka
date: 2020-05-23
categories: 消息队列
tags: Kafka SpringBoot
author: 龙德
---

* content
{:toc}

源码地址：[https://github.com/miansen/kafka-example/tree/master/kafka-example-springboot](https://github.com/miansen/kafka-example/tree/master/kafka-example-springboot)

在 Spring Boot 中使用 Kafka 跟在 Spring MVC 中是差不多的，而且还更简单，不需要各种 xml 配置了，只需要添加注解就可以了。

以下是我写的简单的实例代码，看一下就知道怎么使用了，感觉都是套路。

## application.properties

首先是配置文件，这是必不可少的。Spring Boot 会自动加载配置文件的参数，按照你设置的参数初始化好 kafka 相关的 bean，注入到 IOC 容器里。

```properties
server.port=8080
spring.kafka.consumer.bootstrap-servers=192.168.197.6:9092
# 确保新的消费者组能获得我们之前发送的消息，为了测试方便（生产配置latest，只获取最新的消息）。
spring.kafka.consumer.auto-offset-reset=earliest
spring.kafka.producer.bootstrap-servers=192.168.197.6:9092
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.apache.kafka.common.serialization.StringSerializer

# 监听的 topic 如果不存在，则不报错
spring.kafka.listener.missing-topics-fatal=false
```

以上几个参数是比较常用的，但是我们知道 Kafka 客户端提供的参数配置不止这些，那么如何在 Spring Boot 中配置呢？

熟悉 Spring Boot 套路的同学可能就知道了，Spring Boot 一般都会提供一个 xxxProperties 的配置类，你在配置文件里配置的参数都会映射都这个配置类里。

所以直接搜索 `KafkaProperties` 类查看它有哪些属性就知道 Spring Boot 支持哪些配置参数了。

![image](https://miansen.wang/assets/20200620183631.png)

KafkaProperties 类的属性都写了注释，结合 Kafka 官方文档 [http://kafka.apachecn.org/documentation.html#producerapi](http://kafka.apachecn.org/documentation.html#producerapi) 就可以配置了。

## 生产者

```java
@Service
public class KafkaProducerService {

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;
    
    public void sendMessage(String topic, String value) {
        sendMessage(topic, null, value);
    }
    
    public void sendMessage(String topic, String key, String value) {
        ListenableFuture<SendResult<String, String>> future = kafkaTemplate.send(topic, key, value);
        future.addCallback(success -> {
            RecordMetadata metadata = success.getRecordMetadata();
            System.out.println("生产者发送消息成功。topic: " + metadata.topic() + ", partition: " + metadata.partition() + ", offset: " + metadata.offset());
        }, failure -> {
            System.out.println("生产者发送消息失败，原因：" + failure.getMessage());
        });
    }
    
}
```

还记得在 Spring MVC 里，KafkaTemplate 是需要我们手动配置的吗，在 Spring Boot 里是自动配置的，可以直接拿来使用。

## 消费者

还记得在 Spring MVC 中是怎么配置消费者的吗？首先配置监听类，然后配置监听信息，最后再配置监听容器。

在 Spring Boot 中只需要配置 `@KafkaListener` 注解就可以了。

```java
@Service
public class KafkaConsumerService {

    @KafkaListener(topics = {"test01"}, groupId = "group01")
    public void onMessage1(ConsumerRecord<String, String> record) {
        System.out.println(String.format("[group01-消费者1]收到了消息。topic: %s, partition: %s, offset: %s, key: %s, value: %s",
                record.topic(), record.partition(), record.offset(), record.key(), record.value()));
    }
    
    @KafkaListener(topics = {"test01"}, groupId = "group01")
    public void onMessage2(ConsumerRecord<String, String> record) {
        System.out.println(String.format("[group01-消费者2]收到了消息。topic: %s, partition: %s, offset: %s, key: %s, value: %s",
                record.topic(), record.partition(), record.offset(), record.key(), record.value()));
    }
    
    @KafkaListener(topics = {"test01"}, groupId = "group02")
    public void onMessage3(ConsumerRecord<String, String> record) {
        System.out.println(String.format("[group02-消费者3]收到了消息。topic: %s, partition: %s, offset: %s, key: %s, value: %s",
                record.topic(), record.partition(), record.offset(), record.key(), record.value()));
    }
}
```

我配置了 3 个消费者，它们都消费同一个主题。其中前两个消费者位于同一个消费者组下，第 3 个消费者位于另一个消费者组下。

## 测试

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class ApplicationTest {

    @Autowired
    private KafkaProducerService producerService;

    @Test
    public void sendMessageTest() throws Exception {
        for (int i = 0; i < 10; i++) {
            producerService.sendMessage("test01", Integer.toString(i), "hello kafka-" + i + "-" + new Date().getTime());
        }
        System.in.read(); // 让 main 线程阻塞一下，否则消费者线程可能会来不及消费就死了
    }
    
}

```

运行测试类，可以看到有三条消费者线程在监听消息，分别对应 KafkaConsumerService 类的 3 个 @KafkaListener 注解方法。

![image](https://miansen.wang/assets/20200626113753.png)

控制台输出如下：

```
生产者发送消息成功。topic: test01, partition: 2, offset: 773
生产者发送消息成功。topic: test01, partition: 0, offset: 922
生产者发送消息成功。topic: test01, partition: 2, offset: 774
生产者发送消息成功。topic: test01, partition: 2, offset: 775
生产者发送消息成功。topic: test01, partition: 1, offset: 740
生产者发送消息成功。topic: test01, partition: 0, offset: 923
生产者发送消息成功。topic: test01, partition: 1, offset: 741
生产者发送消息成功。topic: test01, partition: 0, offset: 924
生产者发送消息成功。topic: test01, partition: 0, offset: 925
生产者发送消息成功。topic: test01, partition: 2, offset: 776
[group02-消费者3]收到了消息。topic: test01, partition: 2, offset: 773, key: 0, value: hello kafka-0-1593143698305
[group01-消费者1]收到了消息。topic: test01, partition: 2, offset: 773, key: 0, value: hello kafka-0-1593143698305
[group01-消费者2]收到了消息。topic: test01, partition: 0, offset: 922, key: 1, value: hello kafka-1-1593143713542
[group01-消费者1]收到了消息。topic: test01, partition: 2, offset: 774, key: 2, value: hello kafka-2-1593143744094
[group01-消费者1]收到了消息。topic: test01, partition: 2, offset: 775, key: 3, value: hello kafka-3-1593143746141
[group02-消费者3]收到了消息。topic: test01, partition: 0, offset: 922, key: 1, value: hello kafka-1-1593143713542
[group02-消费者3]收到了消息。topic: test01, partition: 2, offset: 774, key: 2, value: hello kafka-2-1593143744094
[group01-消费者2]收到了消息。topic: test01, partition: 1, offset: 740, key: 4, value: hello kafka-4-1593143746861
[group01-消费者2]收到了消息。topic: test01, partition: 0, offset: 923, key: 5, value: hello kafka-5-1593143746893
[group01-消费者2]收到了消息。topic: test01, partition: 1, offset: 741, key: 6, value: hello kafka-6-1593143746964
[group01-消费者1]收到了消息。topic: test01, partition: 2, offset: 776, key: 9, value: hello kafka-9-1593143747124
[group01-消费者2]收到了消息。topic: test01, partition: 0, offset: 924, key: 7, value: hello kafka-7-1593143747027
[group01-消费者2]收到了消息。topic: test01, partition: 0, offset: 925, key: 8, value: hello kafka-8-1593143747063
[group02-消费者3]收到了消息。topic: test01, partition: 0, offset: 923, key: 5, value: hello kafka-5-1593143746893
[group02-消费者3]收到了消息。topic: test01, partition: 0, offset: 924, key: 7, value: hello kafka-7-1593143747027
[group02-消费者3]收到了消息。topic: test01, partition: 0, offset: 925, key: 8, value: hello kafka-8-1593143747063
[group02-消费者3]收到了消息。topic: test01, partition: 1, offset: 740, key: 4, value: hello kafka-4-1593143746861
[group02-消费者3]收到了消息。topic: test01, partition: 1, offset: 741, key: 6, value: hello kafka-6-1593143746964
[group02-消费者3]收到了消息。topic: test01, partition: 2, offset: 775, key: 3, value: hello kafka-3-1593143746141
[group02-消费者3]收到了消息。topic: test01, partition: 2, offset: 776, key: 9, value: hello kafka-9-1593143747124
```

可以看到生产者生产了 10 条消息，均匀的分布在 3 个分区里。

"消费者1" 只消费了 2 号分区，而"消费者2" 消费了 0 号和 1 号分区。由于这两个消费者位于同一个消费者组下，所以不会重复消费同一个分区。

而 "消费者3" 位于不同的消费者组下，其它组对它不影响，跟它没关系。并且这个组只有它一个消费者，所以 "消费者3" 可以大饱口福，消费所有的分区。