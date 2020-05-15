---
layout: post
title: 17.学习ActiveMQ-在SpringBoot中使用ActiveMQ
date: 2020-05-15
categories: ActiveMQ
tags: ActiveMQ SpringBoot
author: 龙德
---

* content
{:toc}

## 新建工程

新建一个 Maven 工程，我的工程叫 springboot-activemq-example。

pom.xml 文件如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.8.RELEASE</version>
        <relativePath/>
    </parent>
    
    <groupId>wang.miansen</groupId>
    <artifactId>springboot-activemq-example</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>
    <name>springboot-activemq-example</name>
    <description>activemq project for Spring Boot</description>
    
    <properties>
        <java.version>1.8</java.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <maven-jar-plugin.version>3.1.1</maven-jar-plugin.version>
    </properties>

    <dependencies>
    
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-activemq</artifactId>
        </dependency>
        
        <!--消息队列连接池-->
        <dependency>
            <groupId>org.apache.activemq</groupId>
            <artifactId>activemq-pool</artifactId>
        </dependency>
        
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```

## application.properties

```properties
server.port=8080
# 配置消息的类型，如果是true则表示为topic消息，如果为false表示Queue消息
spring.jms.pub-sub-domain=true
# 连接用户名
spring.activemq.user=admin
# 连接密码
spring.activemq.password=admin
# 连接主机信息
spring.activemq.brokerUrl=tcp://localhost:61616
# 连接池的最大连接数
jms.pool.maxConnections=10
```

## 新建配置类 ActiveMQConfig

```java
@Configuration
@EnableJms // 开启JMS适配的注解
public class ActiveMQConfig {

    // 队列1
    @Bean(name = "springboot-activemq-queue-1")
    public Queue queue1() {
        return new ActiveMQQueue("springboot-activemq-queue-1");
    }
    
    // 队列2
    @Bean(name = "springboot-activemq-queue-2")
    public Queue queue2() {
        return new ActiveMQQueue("springboot-activemq-queue-2");
    }

    // 主题1
    @Bean(name = "springboot-activemq-topic-1")
    public Topic topic1() {
        return new ActiveMQTopic("springboot-activemq-topic-1");
    }
    
    // 主题2
    @Bean(name = "springboot-activemq-topic-2")
    public Topic topic2() {
        return new ActiveMQTopic("springboot-activemq-topic-2");
    }
    
}
```

## 队列生产者

```java
@Component
public class QueueProducer {

    @Autowired
    private JmsMessagingTemplate jmsMessagingTemplate;
    
    @Resource(name = "springboot-activemq-queue-1")
    private Queue queue1;
    
    @Resource(name = "springboot-activemq-queue-2")
    private Queue queue2;
    
    // 发送消息到队列1
    public void sendQueueMessage1(final Map<String, Object> mapMessage) {
        jmsMessagingTemplate.convertAndSend(queue1, mapMessage);
    }
    
    // 发送消息到队列2
    public void sendQueueMessage2(final Map<String, Object> mapMessage) {
        jmsMessagingTemplate.convertAndSend(queue2, mapMessage);
    }

}
```

## 队列消费者

```java
@Component
public class QueueConsumer {

    // 监听队列1的消息
    @JmsListener(destination = "springboot-activemq-queue-1")
    public void receiveQueueMessage1(Message message) {
        if (message instanceof MapMessage) {
            MapMessage mapMessage = (MapMessage) message;
            try {
                System.out.println("队列消费者1收到了消息：" + mapMessage.getString("message"));
            } catch (JMSException e) {
                e.printStackTrace();
            }
        }
    }

    // 监听队列2的消息
    @JmsListener(destination = "springboot-activemq-queue-2")
    public void receiveQueueMessage2(Message message) {
        if (message instanceof MapMessage) {
            MapMessage mapMessage = (MapMessage) message;
            try {
                System.out.println("队列消费者2收到了消息：" + mapMessage.getString("message"));
            } catch (JMSException e) {
                e.printStackTrace();
            }
        }
    }
    
    // 监听队列2的消息
    @JmsListener(destination = "springboot-activemq-queue-2")
    public void receiveQueueMessage3(Message message) {
        if (message instanceof MapMessage) {
            MapMessage mapMessage = (MapMessage) message;
            try {
                System.out.println("队列消费者3收到了消息：" + mapMessage.getString("message"));
            } catch (JMSException e) {
                e.printStackTrace();
            }
        }
    }

}
```

## 主题生产者

```java
@Component
public class TopicProducer {

    @Autowired
    private JmsMessagingTemplate jmsMessagingTemplate;
    
    @Resource(name = "springboot-activemq-topic-1")
    private Topic topic1;
    
    @Resource(name = "springboot-activemq-topic-2")
    private Topic topic2;
    
    // 发送消息到主题1
    public void sendTopicMessage1(final Map<String, Object> mapMessage) {
        jmsMessagingTemplate.convertAndSend(topic1, mapMessage);
    }
    
    // 发送消息到主题2
    public void sendTopicMessage2(final Map<String, Object> mapMessage) {
        jmsMessagingTemplate.convertAndSend(topic2, mapMessage);
    }

}
```

## 主题消费者

```java
@Component
public class TopicConsumer {

    // 监听主题1的消息
    @JmsListener(destination = "springboot-activemq-topic-1")
    public void receiveQueueMessage1(Message message) {
        if (message instanceof MapMessage) {
            MapMessage mapMessage = (MapMessage) message;
            try {
                System.out.println("主题消费者1收到了消息：" + mapMessage.getString("message"));
            } catch (JMSException e) {
                e.printStackTrace();
            }
        }
    }

    // 监听主题2的消息
    @JmsListener(destination = "springboot-activemq-topic-2")
    public void receiveQueueMessage2(Message message) {
        if (message instanceof MapMessage) {
            MapMessage mapMessage = (MapMessage) message;
            try {
                System.out.println("主题消费者2收到了消息：" + mapMessage.getString("message"));
            } catch (JMSException e) {
                e.printStackTrace();
            }
        }
    }
    
    // 监听主题2的消息
    @JmsListener(destination = "springboot-activemq-topic-2")
    public void receiveQueueMessage3(Message message) {
        if (message instanceof MapMessage) {
            MapMessage mapMessage = (MapMessage) message;
            try {
                System.out.println("主题消费者3收到了消息：" + mapMessage.getString("message"));
            } catch (JMSException e) {
                e.printStackTrace();
            }
        }
    }

}
```

## 测试

如果要测试主题，那么需要修改配置 `spring.jms.pub-sub-domain=true`。

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class ApplicationTest {

    @Autowired
    private QueueProducer queueProducer;
    
    @Autowired
    private TopicProducer topicProducer;
    
    @Test
    public void sendQueueMessage1() throws Exception {
        Map<String, Object> map = new HashMap<>();
        map.put("message", "Hello ActiveMQ，我是队列消息1");
        queueProducer.sendQueueMessage1(map);
    }
    
    @Test
    public void sendQueueMessage2() throws Exception {
        Map<String, Object> map = new HashMap<>();
        map.put("message", "Hello ActiveMQ，我是队列消息2");
        queueProducer.sendQueueMessage2(map);
        queueProducer.sendQueueMessage2(map);
        queueProducer.sendQueueMessage2(map);
    }
    
    @Test
    public void sendTopicMessage1() throws Exception {
        Map<String, Object> map = new HashMap<>();
        map.put("message", "Hello ActiveMQ，我是主题消息1");
        topicProducer.sendTopicMessage1(map);
    }
    
    @Test
    public void sendTopicMessage2() throws Exception {
        Map<String, Object> map = new HashMap<>();
        map.put("message", "Hello ActiveMQ，我是主题消息2");
        topicProducer.sendTopicMessage2(map);
    }
    
}
```

## 同时配置队列和主题

上面说过，如果想要使用主题，那么需要修改配置文件，这样不方便。我们可以在 ActiveMQConfig 类里同时配置队列和主题。

### 修改 ActiveMQConfig 类：

```java
@Configuration
@EnableJms // 开启JMS适配的注解
public class ActiveMQConfig {

    @Value("${spring.activemq.brokerUrl}")
    private String brokerUrl;

    @Value("${spring.activemq.user}")
    private String user;

    @Value("${spring.activemq.password}")
    private String password;

    @Value("${jms.pool.maxConnections}")
    private int maxConnections;

    // 队列1
    @Bean(name = "springboot-activemq-queue-1")
    public Queue queue1() {
        return new ActiveMQQueue("springboot-activemq-queue-1");
    }

    // 队列2
    @Bean(name = "springboot-activemq-queue-2")
    public Queue queue2() {
        return new ActiveMQQueue("springboot-activemq-queue-2");
    }

    // 主题1
    @Bean(name = "springboot-activemq-topic-1")
    public Topic topic1() {
        return new ActiveMQTopic("springboot-activemq-topic-1");
    }

    // 主题2
    @Bean(name = "springboot-activemq-topic-2")
    public Topic topic2() {
        return new ActiveMQTopic("springboot-activemq-topic-2");
    }

    // 真正可以产生连接的工厂
    @Bean
    public ActiveMQConnectionFactory connectionFactory() {
        return new ActiveMQConnectionFactory(user, password, brokerUrl);
    }

    // ActiveMQ 提供的连接池
    @Bean
    public PooledConnectionFactory pooledConnectionFactory(ActiveMQConnectionFactory connectionFactory) {
        PooledConnectionFactory pool = new PooledConnectionFactory();
        pool.setConnectionFactory(connectionFactory);
        pool.setMaxConnections(maxConnections);
        return pool;
    }

    // 队列监听容器
    @Bean
    public JmsListenerContainerFactory<?> jmsListenerContainerQueue(PooledConnectionFactory connectionFactory) {
        DefaultJmsListenerContainerFactory factory = new DefaultJmsListenerContainerFactory();
        factory.setPubSubDomain(false);
        factory.setConnectionFactory(connectionFactory);
        return factory;
    }

    // 主题监听容器
    @Bean
    public JmsListenerContainerFactory<?> jmsListenerContainerTopic(PooledConnectionFactory connectionFactory) {
        DefaultJmsListenerContainerFactory factory = new DefaultJmsListenerContainerFactory();
        factory.setPubSubDomain(true);
        factory.setConnectionFactory(connectionFactory);
        return factory;
    }

    // 主题监听容器（持久化订阅）
    @Bean
    public JmsListenerContainerFactory<?> jmsListenerContainerSubscriptionDurable1(
            PooledConnectionFactory connectionFactory) {
        DefaultJmsListenerContainerFactory factory = new DefaultJmsListenerContainerFactory();
        factory.setPubSubDomain(true);
        factory.setConnectionFactory(connectionFactory);
        // 持久化订阅
        factory.setSubscriptionDurable(true);
        // 持久订阅必须指定一个 clientId
        factory.setClientId("springboot-activemq-topic-clientId-1");
        return factory;
    }

    // 主题监听容器（持久化订阅）
    @Bean
    public JmsListenerContainerFactory<?> jmsListenerContainerSubscriptionDurable2(
            PooledConnectionFactory connectionFactory) {
        DefaultJmsListenerContainerFactory factory = new DefaultJmsListenerContainerFactory();
        factory.setPubSubDomain(true);
        factory.setConnectionFactory(connectionFactory);
        // 持久化订阅
        factory.setSubscriptionDurable(true);
        // 持久订阅必须指定一个 clientId
        factory.setClientId("springboot-activemq-topic-clientId-2");
        return factory;
    }
    
    @Bean
    public JmsMessagingTemplate jmsMessagingTemplate(PooledConnectionFactory connectionFactory){
        return new JmsMessagingTemplate(connectionFactory);
    }

}
```

### 修改队列消费者

```java
@Component
public class QueueConsumer {

    // 监听队列1的消息
    @JmsListener(destination = "springboot-activemq-queue-1", containerFactory = "jmsListenerContainerQueue")
    public void receiveQueueMessage1(Message message) {
        if (message instanceof MapMessage) {
            MapMessage mapMessage = (MapMessage) message;
            try {
                System.out.println("队列消费者1收到了消息：" + mapMessage.getString("message"));
            } catch (JMSException e) {
                e.printStackTrace();
            }
        }
    }

    // 监听队列2的消息
    @JmsListener(destination = "springboot-activemq-queue-2", containerFactory = "jmsListenerContainerQueue")
    public void receiveQueueMessage2(Message message) {
        if (message instanceof MapMessage) {
            MapMessage mapMessage = (MapMessage) message;
            try {
                System.out.println("队列消费者2收到了消息：" + mapMessage.getString("message"));
            } catch (JMSException e) {
                e.printStackTrace();
            }
        }
    }

    // 监听队列2的消息
    @JmsListener(destination = "springboot-activemq-queue-2", containerFactory = "jmsListenerContainerQueue")
    public void receiveQueueMessage3(Message message) {
        if (message instanceof MapMessage) {
            MapMessage mapMessage = (MapMessage) message;
            try {
                System.out.println("队列消费者3收到了消息：" + mapMessage.getString("message"));
            } catch (JMSException e) {
                e.printStackTrace();
            }
        }
    }

}
```

### 修改主题消费者

```java
@Component
public class TopicConsumer {

    // 监听主题1的消息
    @JmsListener(destination = "springboot-activemq-topic-1", containerFactory = "jmsListenerContainerTopic")
    public void receiveQueueMessage1(Message message) {
        if (message instanceof MapMessage) {
            MapMessage mapMessage = (MapMessage) message;
            try {
                System.out.println("主题消费者1收到了消息：" + mapMessage.getString("message"));
            } catch (JMSException e) {
                e.printStackTrace();
            }
        }
    }

    // 监听主题2的消息
    @JmsListener(destination = "springboot-activemq-topic-2", containerFactory = "jmsListenerContainerSubscriptionDurable1")
    public void receiveQueueMessage2(Message message) {
        if (message instanceof MapMessage) {
            MapMessage mapMessage = (MapMessage) message;
            try {
                System.out.println("主题消费者2收到了消息：" + mapMessage.getString("message"));
            } catch (JMSException e) {
                e.printStackTrace();
            }
        }
    }

    // 监听主题2的消息
    @JmsListener(destination = "springboot-activemq-topic-2", containerFactory = "jmsListenerContainerSubscriptionDurable2")
    public void receiveQueueMessage3(Message message) {
        if (message instanceof MapMessage) {
            MapMessage mapMessage = (MapMessage) message;
            try {
                System.out.println("主题消费者3收到了消息：" + mapMessage.getString("message"));
            } catch (JMSException e) {
                e.printStackTrace();
            }
        }
    }

}
```

<span id="issueId" style="display: none;">5</span>