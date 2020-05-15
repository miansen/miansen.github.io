---
layout: post
title: 16.学习ActiveMQ-在SpringMVC中使用ActiveMQ
date: 2020-05-13
categories: ActiveMQ
tags: ActiveMQ SpringMVC
author: 龙德
---

* content
{:toc}

## 新建工程

新建一个 Maven 工程，我的工程叫 springmvc-activemq-example。

pom.xml 文件如下：

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>wang.miansen</groupId>
    <artifactId>springmvc-activemq-example</artifactId>
    <packaging>war</packaging>
    <version>0.0.1-SNAPSHOT</version>
    <name>springmvc-activemq-example Maven Webapp</name>
    <url>http://maven.apache.org</url>

    <properties>
        <java.version>1.8</java.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    </properties>

    <dependencies>
    
        <!-- 单元测试 -->
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.11</version>
        </dependency>
        
        <!-- 1)Spring核心 -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-core</artifactId>
            <version>4.1.7.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-beans</artifactId>
            <version>4.1.7.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>4.1.7.RELEASE</version>
        </dependency>
        <!-- 2)Spring DAO层 -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-jdbc</artifactId>
            <version>4.1.7.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-tx</artifactId>
            <version>4.1.7.RELEASE</version>
        </dependency>
        <!-- 3)Spring web -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-web</artifactId>
            <version>4.1.7.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-webmvc</artifactId>
            <version>4.1.7.RELEASE</version>
        </dependency>
        <!-- 4)Spring test -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-test</artifactId>
            <version>4.1.7.RELEASE</version>
        </dependency>

        <!-- 1)ActiveMQ 核心 -->
        <dependency>
            <groupId>org.apache.activemq</groupId>
            <artifactId>activemq-core</artifactId>
            <version>5.7.0</version>
        </dependency>

        <!-- 2)ActiveMQ与 Spring 集成 -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-jms</artifactId>
            <version>4.3.23.RELEASE</version>
        </dependency>

        <!--3)ActiveMQ所需要的pool包 -->
        <dependency>
            <groupId>org.apache.activemq</groupId>
            <artifactId>activemq-pool</artifactId>
            <version>5.15.9</version>
        </dependency>
    </dependencies>

    <build>
        <finalName>springmvc-activemq-example</finalName>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <!-- web.xml版本 -->
                <version>3.1</version>
                <configuration>
                    <!-- jdk版本 -->
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

## 队列生产者

新建队列生产者 QueueProducer 类：

```java
public class QueueProducer {

    private JmsTemplate jmsTemplate;
    
    public void sendMessage(final Map<String, Object> mapMessage) {
        jmsTemplate.convertAndSend(mapMessage);
    }

    public void setJmsTemplate(JmsTemplate jmsTemplate) {
        this.jmsTemplate = jmsTemplate;
    }
    
}
```

## 队列消费者

新建 3 个队列消费者，这是其中一个，其它两个都一样的，就不贴出来了。

```java
public class QueueConsumer1 implements MessageListener {

    @Override
    public void onMessage(Message message) {
        if (message instanceof MapMessage) {
            MapMessage mapMessage = (MapMessage) message;
            try {
                System.out.println("队列消费者1收到了消息：" + mapMessage.getString("message"));
            } catch (JMSException e) {
                e.printStackTrace();
            }
        }
    }

}
```

## 主题生产者

新建主题生产者 TopicProducer 类：

```java
public class TopicProducer {

    private JmsTemplate jmsTemplate;
    
    public void sendMessage(final Topic topic, final Map<String, Object> mapMessage) {
        jmsTemplate.convertAndSend(topic, mapMessage);
    }

    public void setJmsTemplate(JmsTemplate jmsTemplate) {
        this.jmsTemplate = jmsTemplate;
    }
    
}
```

## 主题消费者

新建 3 个主题消费者，这是其中一个，其它两个都一样的，就不贴出来了。

```java
public class TopicConsumer1 implements MessageListener {

    @Override
    public void onMessage(Message message) {
        if (message instanceof MapMessage) {
            MapMessage mapMessage = (MapMessage) message;
            try {
                System.out.println("主题消费者1收到了消息：" + mapMessage.getString("message"));
            } catch (JMSException e) {
                e.printStackTrace();
            }
        }
    }

}
```

## application.properties

新建 application.properties 文件，配置 ActiveMQ 的相关信息。

```properties
# jms服务器地址
jms.broker.url=tcp://localhost:61616
# 连接用户名
jms.broker.userName=admin
# 连接密码
jms.broker.password=admin
# 使用异步发送
jms.broker.useAsyncSend=true
# 连接池的最大连接数
jms.pool.maxConnections=10
```

## application-context.xml

新建 application-context.xml 文件，配置各个 Bean。

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

    <!-- 连接工厂 -->
    <bean id="activeMQConnectionFactory" class="org.apache.activemq.ActiveMQConnectionFactory">
        <property name="brokerURL" value="${jms.broker.url}"/>
        <property name="userName" value="${jms.broker.userName}"></property>  
        <property name="password" value="${jms.broker.password}"></property>  
        <property name="useAsyncSend" value="${jms.broker.useAsyncSend}" />  
    </bean>
    
    <!-- 连接池 -->
    <!-- ActiveMQ 为我们提供了一个 PooledConnectionFactory，通过往里面注入一个 ActiveMQConnectionFactory   
        可以用来将 Connection、Session 和  MessageProducer 池化，这样可以大大的减少我们的资源消耗，要依赖于 activemq-pool 包 -->  
    <bean id="pooledConnectionFactory" class="org.apache.activemq.pool.PooledConnectionFactory">
        <!--真正可以产生连接的连接工厂，由对应的 JMS 服务厂商提供-->
        <property name="connectionFactory" ref="activeMQConnectionFactory" />
        <!--连接池的最大连接数-->
        <property name="maxConnections" value="${jms.pool.maxConnections}"/>
    </bean>
    
    <!-- Spring 用于管理真正的 ConnectionFactory 的 ConnectionFactory -->
    <!-- 队列和非持久化订阅使用 -->
    <bean id="singleConnectionFactory" class="org.springframework.jms.connection.SingleConnectionFactory">
        <!-- 目标 ConnectionFactory，对应真实的可以产生 JMS Connection 的 ConnectionFactory -->  
        <property name="targetConnectionFactory" ref="pooledConnectionFactory"/>
    </bean>
    
    <!-- Spring 用于管理真正的 ConnectionFactory 的 ConnectionFactory -->
    <!-- 持久化订阅使用 -->
    <bean id="subscriptionDurableSingleConnectionFactory2" class="org.springframework.jms.connection.SingleConnectionFactory">
        <!-- 目标 ConnectionFactory，对应真实的可以产生 JMS Connection 的 ConnectionFactory -->  
        <property name="targetConnectionFactory" ref="pooledConnectionFactory"/>
         <!--持久订阅ID -->  
        <property name="clientId" value="springmvc-activemq-topic-clientId-2" />
    </bean>
    
    <!-- Spring 用于管理真正的 ConnectionFactory 的 ConnectionFactory -->
    <!-- 持久化订阅使用 -->
    <bean id="subscriptionDurableSingleConnectionFactory3" class="org.springframework.jms.connection.SingleConnectionFactory">
        <!-- 目标 ConnectionFactory，对应真实的可以产生 JMS Connection 的 ConnectionFactory -->  
        <property name="targetConnectionFactory" ref="pooledConnectionFactory"/>
         <!--持久订阅ID -->  
        <property name="clientId" value="springmvc-activemq-topic-clientId-3" />
    </bean>
    
    <!--队列1-->
    <bean id="activeMQQueue1" class="org.apache.activemq.command.ActiveMQQueue">
        <!-- 队列的名称 -->
        <constructor-arg index="0" value="springmvc-activemq-queue-1" />
    </bean>
    
    <!--队列2-->
    <bean id="activeMQQueue2" class="org.apache.activemq.command.ActiveMQQueue">
        <!-- 队列的名称 -->
        <constructor-arg index="0" value="springmvc-activemq-queue-2" />
    </bean>
    
    <!--主题1-->
    <bean id="activeMQTopic1" class="org.apache.activemq.command.ActiveMQTopic">
        <!-- 主题的名称 -->
        <constructor-arg index="0" value="springmvc-activemq-topic-1"/>
    </bean>
    
    <!--主题2-->
    <bean id="activeMQTopic2" class="org.apache.activemq.command.ActiveMQTopic">
        <!-- 主题的名称 -->
        <constructor-arg index="0" value="springmvc-activemq-topic-2"/>
    </bean>
    
    <!-- 队列 JMS 模板，Spring 提供的 JMS 工具类，用它发送、接收消息。 -->
    <bean id="jmsTemplateQueue" class="org.springframework.jms.core.JmsTemplate">
        <!-- 连接池 -->
        <property name="connectionFactory" ref="singleConnectionFactory"/>
        <!-- 默认的目的地，如果发送时不指定目的地，那么就用这个默认的目的地 -->
        <property name="defaultDestination" ref="activeMQQueue1"/>
        <!-- 消息转换器 -->
        <property name="messageConverter" ref="simpleMessageConverter" />
        <!-- 发送模式，1: 非持久化，2: 持久化 -->
        <property name="deliveryMode" value="2" />
        <!-- 是否为发布订阅模式，队列是 false，主题是 true --> 
        <property name="pubSubDomain" value="false"/>
    </bean>
    
    <!-- 主题 JMS 模板，Spring 提供的 JMS 工具类，用它发送、接收消息。 -->
    <bean id="jmsTemplateTopic" class="org.springframework.jms.core.JmsTemplate">
        <!-- 连接池 -->
        <property name="connectionFactory" ref="singleConnectionFactory"/>
        <!-- 默认的目的地，如果发送时不指定目的地，那么就用这个默认的目的地 -->
        <property name="defaultDestination" ref="activeMQTopic1"/>
         <!-- 消息转换器 -->
        <property name="messageConverter" ref="simpleMessageConverter" />
        <!-- 发送模式，1: 非持久化，2: 持久化 -->
        <property name="deliveryMode" value="2" />
        <!-- 是否为发布订阅模式，队列是 false，主题是 true --> 
        <property name="pubSubDomain" value="true"/>
    </bean>
    
    <!--队列监听容器1，一经注册，自动监听-->
    <bean id="queueListenerContainer1" class="org.springframework.jms.listener.DefaultMessageListenerContainer">
        <!-- 连接池 -->
        <property name="connectionFactory" ref="singleConnectionFactory" />
        <!-- 是否为发布订阅模式，队列是 false，主题是 true --> 
        <property name="pubSubDomain" value="false" />
        <!-- 监听目的地 -->
        <property name="destination" ref="activeMQQueue1" />
        <!-- 监听类 -->
        <property name="messageListener" ref="queueConsumer1" />
    </bean>
    
    <!--队列监听容器2，一经注册，自动监听-->
    <bean id="queueListenerContainer2" class="org.springframework.jms.listener.DefaultMessageListenerContainer">
        <!-- 连接池 -->
        <property name="connectionFactory" ref="singleConnectionFactory" />
        <!-- 是否为发布订阅模式，队列是 false，主题是 true --> 
        <property name="pubSubDomain" value="false" />
        <!-- 监听目的地 -->
        <property name="destination" ref="activeMQQueue2" />
        <!-- 监听类 -->
        <property name="messageListener" ref="queueConsumer2" />
    </bean>
    
    <!--队列监听容器3，注意目的地跟容器2一样-->
    <bean id="queueListenerContainer3" class="org.springframework.jms.listener.DefaultMessageListenerContainer">
        <!-- 连接池 -->
        <property name="connectionFactory" ref="singleConnectionFactory" />
        <!-- 是否为发布订阅模式，队列是 false，主题是 true --> 
        <property name="pubSubDomain" value="false" />
        <!-- 监听目的地 -->
        <property name="destination" ref="activeMQQueue2" />
        <!-- 监听类 -->
        <property name="messageListener" ref="queueConsumer3" />
    </bean>
    
    <!--主题监听容器1（非持久化订阅），一经注册，自动监听-->
    <bean id="topicListenerContainer1" class="org.springframework.jms.listener.DefaultMessageListenerContainer">
        <!-- 连接池 -->
        <property name="connectionFactory" ref="singleConnectionFactory" />
        <!-- 监听目的地 -->
        <property name="destination" ref="activeMQTopic1" />
        <!-- 持久化订阅 -->
        <property name="subscriptionDurable" value="false" />
        <!-- 是否为发布订阅模式，队列是 false，主题是 true --> 
        <property name="pubSubDomain" value="true" />
        <!-- 监听类 -->
        <property name="messageListener" ref="topicConsumer1" />
    </bean>
    
    <!--主题监听容器2（持久化订阅），一经注册，自动监听-->
    <bean id="topicListenerContainer2" class="org.springframework.jms.listener.DefaultMessageListenerContainer">
        <!-- 连接池 -->
        <property name="connectionFactory" ref="subscriptionDurableSingleConnectionFactory2" />
        <!-- 监听目的地 -->
        <property name="destination" ref="activeMQTopic2" />
        <!-- 持久化订阅 -->
        <property name="subscriptionDurable" value="true" />
        <!-- 设置接收客户端的 ID，配置持久订阅必须指定一个 clientId -->
        <property name="clientId" value="springmvc-activemq-topic-clientId-2" />
        <!-- 是否为发布订阅模式，队列是 false，主题是 true --> 
        <property name="pubSubDomain" value="true" />
        <!-- 监听类 -->
        <property name="messageListener" ref="topicConsumer2" />
    </bean>
    
    <!-- 主题监听容器3（持久化订阅），注意目的地跟容器2一样 -->
    <bean id="topicListenerContainer3" class="org.springframework.jms.listener.DefaultMessageListenerContainer">
        <!-- 连接池 -->
        <property name="connectionFactory" ref="subscriptionDurableSingleConnectionFactory3" />
        <!-- 监听目的地 -->
        <property name="destination" ref="activeMQTopic2" />
        <!-- 持久化订阅 -->
        <property name="subscriptionDurable" value="true" />
        <!-- 设置接收客户端的 ID，配置持久订阅必须指定一个 clientId -->
        <property name="clientId" value="springmvc-activemq-topic-clientId-3" />
        <!-- 是否为发布订阅模式，队列是 false，主题是 true --> 
        <property name="pubSubDomain" value="true" />
        <!-- 监听类 -->
        <property name="messageListener" ref="topicConsumer3" />
    </bean>
    
    <!-- 队列生产者 -->
    <bean id="queueProducer" class="com.example.activemq.QueueProducer">
        <property name="jmsTemplate" ref="jmsTemplateQueue" />
    </bean>
    
    <!-- 主题生产者 -->
    <bean id="topicProducer" class="com.example.activemq.TopicProducer">
        <property name="jmsTemplate" ref="jmsTemplateTopic" />
    </bean>
    
    <!-- 队列消费者1 -->
    <bean id="queueConsumer1" class="com.example.activemq.QueueConsumer1" />
    <!-- 队列消费者2 -->
    <bean id="queueConsumer2" class="com.example.activemq.QueueConsumer2" />
    <!-- 队列消费者3 -->
    <bean id="queueConsumer3" class="com.example.activemq.QueueConsumer3" />
    
    <!-- 主题消费者1 -->
    <bean id="topicConsumer1" class="com.example.activemq.TopicConsumer1" />
    <!-- 主题消费者2 -->
    <bean id="topicConsumer2" class="com.example.activemq.TopicConsumer2" />
    <!-- 主题消费者3 -->
    <bean id="topicConsumer3" class="com.example.activemq.TopicConsumer3" />
    
    <!-- 消息转换器 -->
    <bean id="simpleMessageConverter" class="org.springframework.jms.support.converter.SimpleMessageConverter" />
</beans>
```

## 项目结构

最后的项目结构如下：

![image](https://miansen.wang/assets/20200515102216.png)

## 测试

新建测试类 ApplicationTest

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration({"classpath:application-context.xml"})
public class ApplicationTest {

    @Resource(name = "activeMQQueue1")
    private Queue queue1;

    @Resource(name = "activeMQQueue2")
    private Queue queue2;

    @Resource(name = "activeMQTopic1")
    private Topic topic1;

    @Resource(name = "activeMQTopic2")
    private Topic topic2;

    @Autowired
    private QueueProducer queueProducer;

    @Autowired
    private TopicProducer topicProducer;

    private Map<String, Object> mapMessage;

    @Before
    public void initMessage() throws Exception {
        mapMessage = new HashMap<>();
        mapMessage.put("message", "Hello ActiveMQ - " + new Date().getTime());
    }

    // 发送消息到队列1
    @Test
    public void queue1() throws Exception {
        queueProducer.sendMessage(queue1, mapMessage);
    }

    // 发送消息到队列2
    @Test
    public void queue2() throws Exception {
        queueProducer.sendMessage(queue2, mapMessage);
    }

    // 发送消息到主题1
    @Test
    public void topic1() throws Exception {
        topicProducer.sendMessage(topic1, mapMessage);
    }

    // 发送消息到主题2
    @Test
    public void topic2() throws Exception {
        topicProducer.sendMessage(topic2, mapMessage);
    }

}
```

### 测试队列1

运行 queue1() 方法，生产者发送消息后，消费者通过监听的方式收到了消息。

![image](https://miansen.wang/assets/20200515103413.png)

### 测试队列2

由于队列2被两个消费者监听，又由于 ActiveMQ 队列中的消息只能被一个消费者消费，所以这两个消费者应该是通过负载均衡的方式消费。

运行 queue2() 方法，可以看到有时是消费者2消费，有时又是消费者3消费。

## 测试主题1

运行 topic1() 方法，生产者发送消息后，消费者通过监听的方式收到了消息。

## 测试主题2

运行 topic2() 方法，生产者发送消息后，由于是发布订阅的模式，所以消费者2和消费者3都能收到消息。

查看控制台，可以看到有两个持久化订阅的消费者。

![image](https://miansen.wang/assets/20200515104134.png)

<span id="issueId" style="display: none;">4</span>