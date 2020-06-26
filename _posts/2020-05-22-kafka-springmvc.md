---
layout: post
title: 4.学习 Kafka - 在 Spring MVC 中使用 Kafka
date: 2020-05-22
categories: Kafka
tags: Kafka SpringMVC
author: 龙德
---

* content
{:toc}

源码地址：[https://github.com/miansen/kafka-example/tree/master/kafka-example-springmvc](https://github.com/miansen/kafka-example/tree/master/kafka-example-springmvc)

## 引入依赖

Spring MVC 的依赖就不贴了，可以去源码里看。

既然是在 Spring MVC 应用中使用 Kafka，那肯定会有一个 spring-kafka 的玩意，一般 Spring 应用都是通过这种方式将第三方应用集成进来，简化开发，开箱即用。

```xml
<!-- 1)Kafka与 Spring 集成 -->
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
    <version>1.0.5.RELEASE</version>
</dependency>
```
要注意 spring-kafka 与 Spring MVC 的版本，太高了可能会不兼容。

## application.properties

application.properties 文件配置 kafka 用到的各个属性，这样就不用再代码里或者是 xml 配置文件里写死了，更灵活一点。

```properties
bootstrap.servers=192.168.197.6:9092
acks=all
retries=0
batch.size=16384
linger.ms=1
buffer.memory=33554432
key.serializer=org.apache.kafka.common.serialization.StringSerializer
value.serializer=org.apache.kafka.common.serialization.StringSerializer
key.deserializer=org.apache.kafka.common.serialization.StringDeserializer
value.deserializer=org.apache.kafka.common.serialization.StringDeserializer
group.id=my-consumer-group-04
session.timeout.ms=15000
enable.auto.commit=true
auto.commit.interval.ms=1000
```

各个属性的作用可以在这里看：[http://kafka.apachecn.org/documentation.html](http://kafka.apachecn.org/documentation.html)

## 配置文件

既然是 Spring MVC 应用，那么 xml 配置文件也是必不可少的。虽然可以通过 Java Config 的形式配置，但是还是用 xml 配置比较好，这样直观一点，可以更明显的看出配置了哪个 bean，哪个 bean 跟 哪个 bean 关联。

我分两个配置文件，一个是 kafka-producer.xml，这个是配置生产者的。还有一个是 kafka-consumer.xml，这个是配置消费者的。

kafka-producer.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:context="http://www.springframework.org/schema/context"
    xmlns:tx="http://www.springframework.org/schema/tx"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
    http://www.springframework.org/schema/beans/spring-beans.xsd
    http://www.springframework.org/schema/context
    http://www.springframework.org/schema/context/spring-context.xsd
    http://www.springframework.org/schema/tx
    http://www.springframework.org/schema/tx/spring-tx.xsd">

    <!-- 读取配置文件 -->
    <bean id="propertyPlaceholderConfigurer" class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
        <property name="locations">
            <list>
                <value>classpath:application.properties</value>
            </list>
        </property>
    </bean>

     <!-- 配置生产者的属性 -->  
     <bean id="kafkaProducerProperties" class="java.util.HashMap">  
        <constructor-arg>  
            <map>  
                <entry key="bootstrap.servers" value="${bootstrap.servers}"/>
                <entry key="acks" value="${acks}"/>
                <entry key="retries" value="${retries}"/>  
                <entry key="batch.size" value="${batch.size}"/>  
                <entry key="linger.ms" value="${linger.ms}"/>  
                <entry key="buffer.memory" value="${buffer.memory}"/>  
                <entry key="key.serializer" value="${key.serializer}"/>  
                <entry key="value.serializer" value="${value.serializer}"/>
                <entry key="key.deserializer" value="${key.deserializer}"/>  
                <entry key="value.deserializer" value="${value.deserializer}"/> 
            </map>  
        </constructor-arg>  
     </bean>  
    
    <!-- 创建 kafkaProducerTemplate 需要使用的 kafkaProducerFactory bean -->
     <bean id="kafkaProducerFactory" class="org.springframework.kafka.core.DefaultKafkaProducerFactory">  
        <constructor-arg ref="kafkaProducerProperties" /> 
     </bean>
     
     <!-- 创建 kafkaTemplate bean，使用的时候，只需要注入这个 bean，即可使用 template 的 send 消息方法 -->
     <bean id="kafkaTemplate" class="org.springframework.kafka.core.KafkaTemplate">  
        <constructor-arg ref="kafkaProducerFactory"/>
        <!--设置对应 topic-->
        <property name="defaultTopic" value="test01"/>
     </bean>
     
    <!-- 生产者 -->
    <bean id="kafkaProducerService" class="wang.miansen.kafka.springmvc.KafkaProducerService" />
</beans>
```

kafka-consumer.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:context="http://www.springframework.org/schema/context"
    xmlns:tx="http://www.springframework.org/schema/tx"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
    http://www.springframework.org/schema/beans/spring-beans.xsd
    http://www.springframework.org/schema/context
    http://www.springframework.org/schema/context/spring-context.xsd
    http://www.springframework.org/schema/tx
    http://www.springframework.org/schema/tx/spring-tx.xsd">

    <!-- 读取配置文件 -->
    <bean id="propertyPlaceholderConfigurer"
        class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
        <property name="locations">
            <list>
                <value>classpath:application.properties</value>
            </list>
        </property>
    </bean>

    <!-- 配置消费者的属性 -->
    <bean id="kafkaConsumerProperties" class="java.util.HashMap">
        <constructor-arg>
            <map>
                <entry key="bootstrap.servers" value="${bootstrap.servers}" />
                <entry key="group.id" value="${group.id}" />
                <entry key="enable.auto.commit" value="${enable.auto.commit}" />
                <entry key="session.timeout.ms" value="${session.timeout.ms}" />
                <entry key="auto.commit.interval.ms" value="${auto.commit.interval.ms}" />
                <entry key="key.serializer" value="${key.serializer}" />
                <entry key="value.serializer" value="${value.serializer}" />
                <entry key="key.deserializer" value="${key.deserializer}" />
                <entry key="value.deserializer" value="${value.deserializer}" />
            </map>
        </constructor-arg>
    </bean>

    <!-- 创建 kafkaConsumerFactory bean -->
    <bean id="kafkaConsumerFactory"
        class="org.springframework.kafka.core.DefaultKafkaConsumerFactory">
        <constructor-arg ref="kafkaConsumerProperties" />
    </bean>

    <!--具体监听的 bean -->
    <bean id="kafkaConsumerListener1" class="wang.miansen.kafka.springmvc.KafkaConsumerListener1" />
    <bean id="kafkaConsumerListener2" class="wang.miansen.kafka.springmvc.KafkaConsumerListener2" />

    <!-- 消费者容器配置信息 -->
    <bean id="kafkaMessageListenerContainerProperties1"
        class="org.springframework.kafka.listener.config.ContainerProperties">
        <!-- 订阅主题 -->
        <constructor-arg value="test01" />
        <property name="messageListener" ref="kafkaConsumerListener1" />
    </bean>
    <bean id="kafkaMessageListenerContainerProperties2"
        class="org.springframework.kafka.listener.config.ContainerProperties">
        <!-- 订阅主题 -->
        <constructor-arg value="test01" />
        <property name="messageListener" ref="kafkaConsumerListener2" />
    </bean>

    <!-- 配置消费者容器 -->
    <bean id="kafkaMessageListenerContainer1"
        class="org.springframework.kafka.listener.KafkaMessageListenerContainer"
        init-method="doStart">
        <constructor-arg ref="kafkaConsumerFactory" />
        <constructor-arg ref="kafkaMessageListenerContainerProperties1" />
    </bean>
    <bean id="kafkaMessageListenerContainer2"
        class="org.springframework.kafka.listener.KafkaMessageListenerContainer"
        init-method="doStart">
        <constructor-arg ref="kafkaConsumerFactory" />
        <constructor-arg ref="kafkaMessageListenerContainerProperties2" />
    </bean>

</beans>
```

## 业务代码

KafkaProducerService

```java
public class KafkaProducerService {

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;
    
    public void sendMessage(String value) {
        sendMessage(null, value);
    }
    
    public void sendMessage(String key, String value) {
        ListenableFuture<SendResult<String, String>> future = kafkaTemplate.sendDefault(value);
        future.addCallback(success -> {
            RecordMetadata metadata = success.getRecordMetadata();
            System.out.println("生产者发送消息成功。topic: " + metadata.topic() + ", partition: " + metadata.partition() + ", offset: " + metadata.offset());
        }, failure -> {
            System.out.println("生产者发送消息失败，原因：" + failure.getMessage());
        });
    }
}
```

KafkaConsumerListener1

```java
public class KafkaConsumerListener1 implements MessageListener<String, String> {

    @Override
    public void onMessage(ConsumerRecord<String, String> record) {
        System.out.println(String.format("消费者1收到了消息。topic: %s, partition: %s, offset: %s, key: %s, value: %s", record.topic(),
                record.partition(), record.offset(), record.key(), record.value()));
    }

}
```

KafkaConsumerListener2

```java
public class KafkaConsumerListener2 implements MessageListener<String, String> {

    @Override
    public void onMessage(ConsumerRecord<String, String> record) {
        System.out.println(String.format("消费者2收到了消息。topic: %s, partition: %s, offset: %s, key: %s, value: %s", record.topic(),
                record.partition(), record.offset(), record.key(), record.value()));
    }

}
```

各个 bean 的作用都写上了注释，看一眼就明白是干啥用的了。如果你仔细看就会发现，在 Spring 应用中使用 Kafka，其实跟你用 kafka-clients 的 API 差不多的，只不过是 Spring 封一层，一般都会提供一个 xxxFactory 和 xxxTemplate 的 Bean，将第三方应用融合进 Spring 应用，以便开发者更容易上手。

在生产者的配置中，我配置了一个 kafkaProducerService bean，这个 bean 是生产者服务的接口，用来处理业务请求，这个 bean 注入了 Spring 提供的操作模板类 kafkaTemplate，这个类有很多操作 Kafka 的方法，通过它可以很方便的将消息发送到 Kafka。

kafkaTemplate 类的 send 方法会返回一个 Future 对象，这个对象接收两个对象，一个用来处理发送成功，一个用来处理发送失败。

在消费者的配置中，我配置了两个监听 bean：kafkaConsumerListener1 和 kafkaConsumerListener2。既然是监听的方式，那么说明消费者是异步消费的，当有消息进来时，会另起线程去处理消费逻辑。

## 测试

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration({"classpath:kafka-producer.xml", "classpath:kafka-consumer.xml"})
public class ApplicationTest {

    @Autowired
    private KafkaProducerService producerService;
    
    @Test
    public void sendMessageTest() throws Exception {
        for (int i = 0; i < 10; i++) {
            producerService.sendMessage(Integer.toString(i), "hello kafka-" + i);
        }
        System.in.read(); // 让 main 线程阻塞一下，否则消费者线程可能会来不及消费就死了
    }
}
```

生产者向 Kafka 发送数据，我这里指定了 key。前面说过，如果不指定分区，但是指定了 key，那么 Kafka 会将 key hash，然后取模，根据模数决定消息分配到哪个分区。所以这 10 条数据会分散到各个分区存储。

控制台输出信息如下，为了方便查看，我截成了两部分。

```
生产者发送消息成功。topic: test01, partition: 0, offset: 880
生产者发送消息成功。topic: test01, partition: 0, offset: 881
生产者发送消息成功。topic: test01, partition: 0, offset: 882
生产者发送消息成功。topic: test01, partition: 1, offset: 714
生产者发送消息成功。topic: test01, partition: 1, offset: 715
生产者发送消息成功。topic: test01, partition: 1, offset: 716
生产者发送消息成功。topic: test01, partition: 2, offset: 731
生产者发送消息成功。topic: test01, partition: 2, offset: 732
生产者发送消息成功。topic: test01, partition: 2, offset: 733
生产者发送消息成功。topic: test01, partition: 2, offset: 734
```

可以看到这 10 条消息均匀的分布在 3 个分区里。

我配置了两个消费者，这两个消费者都位于一个消费者组下，所以这两个消费者会轮流消费，而且只会消费到分配给自己的分区，不会重复消费相同的分区。

![image](https://miansen.wang/assets/20200616171939.png)

可以看到有两条消费者线程在监听处理消息。

```
消费者1收到了消息。topic: test01, partition: 0, offset: 880, key: null, value: hello kafka-2
消费者1收到了消息。topic: test01, partition: 0, offset: 881, key: null, value: hello kafka-5
消费者1收到了消息。topic: test01, partition: 0, offset: 882, key: null, value: hello kafka-8
消费者2收到了消息。topic: test01, partition: 2, offset: 731, key: null, value: hello kafka-0
消费者2收到了消息。topic: test01, partition: 2, offset: 732, key: null, value: hello kafka-3
消费者2收到了消息。topic: test01, partition: 2, offset: 733, key: null, value: hello kafka-6
消费者2收到了消息。topic: test01, partition: 2, offset: 734, key: null, value: hello kafka-9
消费者1收到了消息。topic: test01, partition: 1, offset: 714, key: null, value: hello kafka-1
消费者1收到了消息。topic: test01, partition: 1, offset: 715, key: null, value: hello kafka-4
消费者1收到了消息。topic: test01, partition: 1, offset: 716, key: null, value: hello kafka-7
```

可以看到 "消费者1" 消费了 0 号和 1 号分区，"消费者2" 只消费了 2 号分区。再次强调，这两个消费者位于同一个消费者组下，它们只会消费分配给自己的分区，不会消费他人的分区，也就是说不会重复消费相同的分区。

为了更直观的对比，加深印象，我们再配置一个 "消费者3"，不过这个 "消费者3" 位于不同的消费者组下。

具体配置就不贴出来了，可以到源码里看。

我们直接看结果，控制台输出如下，为了方便查看，我截成了两部分。

```xml
生产者发送消息成功。topic: test01, partition: 0, offset: 887
生产者发送消息成功。topic: test01, partition: 0, offset: 888
生产者发送消息成功。topic: test01, partition: 0, offset: 889
生产者发送消息成功。topic: test01, partition: 1, offset: 720
生产者发送消息成功。topic: test01, partition: 1, offset: 721
生产者发送消息成功。topic: test01, partition: 1, offset: 722
生产者发送消息成功。topic: test01, partition: 1, offset: 723
生产者发送消息成功。topic: test01, partition: 2, offset: 738
生产者发送消息成功。topic: test01, partition: 2, offset: 739
生产者发送消息成功。topic: test01, partition: 2, offset: 740
```

可以看到生产者生产了 10 条消息，均匀的分布在 3 个分区里。

```xml
消费者1收到了消息。topic: test01, partition: 0, offset: 887, key: null, value: hello kafka-1
消费者1收到了消息。topic: test01, partition: 0, offset: 888, key: null, value: hello kafka-4
消费者1收到了消息。topic: test01, partition: 0, offset: 889, key: null, value: hello kafka-7
消费者3收到了消息。topic: test01, partition: 0, offset: 887, key: null, value: hello kafka-1
消费者3收到了消息。topic: test01, partition: 0, offset: 888, key: null, value: hello kafka-4
消费者3收到了消息。topic: test01, partition: 0, offset: 889, key: null, value: hello kafka-7
消费者2收到了消息。topic: test01, partition: 2, offset: 738, key: null, value: hello kafka-2
消费者2收到了消息。topic: test01, partition: 2, offset: 739, key: null, value: hello kafka-5
消费者2收到了消息。topic: test01, partition: 2, offset: 740, key: null, value: hello kafka-8
消费者3收到了消息。topic: test01, partition: 1, offset: 720, key: null, value: hello kafka-0
消费者3收到了消息。topic: test01, partition: 1, offset: 721, key: null, value: hello kafka-3
消费者3收到了消息。topic: test01, partition: 1, offset: 722, key: null, value: hello kafka-6
消费者3收到了消息。topic: test01, partition: 1, offset: 723, key: null, value: hello kafka-9
消费者3收到了消息。topic: test01, partition: 2, offset: 738, key: null, value: hello kafka-2
消费者3收到了消息。topic: test01, partition: 2, offset: 739, key: null, value: hello kafka-5
消费者3收到了消息。topic: test01, partition: 2, offset: 740, key: null, value: hello kafka-8
消费者1收到了消息。topic: test01, partition: 1, offset: 720, key: null, value: hello kafka-0
消费者1收到了消息。topic: test01, partition: 1, offset: 721, key: null, value: hello kafka-3
消费者1收到了消息。topic: test01, partition: 1, offset: 722, key: null, value: hello kafka-6
消费者1收到了消息。topic: test01, partition: 1, offset: 723, key: null, value: hello kafka-9
```

可以看到 "消费者1" 消费了 0 号和 1 号分区，"消费者2" 只消费了 2 号分区。由于这两个消费者位于同一个消费者组下，所以不会重复消费同一个分区。

而 "消费者3" 位于不同的消费者组下，其它组对它不影响，跟它没关系。并且这个组只有它一个消费者，所以 "消费者3" 可以大饱口福，消费所有的分区。