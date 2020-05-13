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

新建生产者 QueueProducer 类：

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

新建消费者 QueueConsumer 类：

```java
public class QueueConsumer implements MessageListener {

    @Override
    public void onMessage(Message message) {
        if (message instanceof MapMessage) {
            MapMessage mapMessage = (MapMessage) message;
            try {
                System.out.println("消费消息：" + mapMessage.getString("name"));
            } catch (JMSException e) {
                e.printStackTrace();
            }
        }
    }

}
```

新建 application.properties 文件，配置 ActiveMQ 的相关信息。

```properties
jms.broker.url = tcp://localhost:61616
```

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

    <!-- 配置连接工厂 -->
    <bean id="activeMQConnectionFactory" class="org.apache.activemq.ActiveMQConnectionFactory">
        <property name="brokerURL" value="${jms.broker.url}"/>
    </bean>
    
    <!-- 配置连接池（spring提供的，一般用这个） -->
    <bean id="cachingConnectionFactory" class="org.springframework.jms.connection.CachingConnectionFactory">
        <!--真正可以产生Connection的ConnectionFactory，由对应的JMS服务厂商提供-->
        <property name="targetConnectionFactory" ref="activeMQConnectionFactory"/>
        <!-- 会话的最大连接数 -->
        <property name="sessionCacheSize" value="100"/>
    </bean>
    
    <!-- 配置连接池（ActiveMQ官方提供的，一般不用这个） -->
    <!-- <bean id="pooledConnectionFactory" class="org.apache.activemq.pool.PooledConnectionFactory" destroy-method="stop">
        <property name="connectionFactory" ref="activeMQConnectionFactory">
        <property name="maxConnections" value="100"/>
    </bean> -->
   
    <!--配置队列目的地-->
    <bean id="defaultDestinationQueue" class="org.apache.activemq.command.ActiveMQQueue">
        <!-- 队列的名称 -->
        <constructor-arg index="0" value="spring-activemq-quque" />
    </bean>

    <!--配置主题目的地-->
    <bean id="defaultDestinationTopic" class="org.apache.activemq.command.ActiveMQTopic">
        <!-- 主题的名称 -->
        <constructor-arg index="0" value="spring-activemq-topic"/>
    </bean>
    
    <!-- 配置队列 JMS 模板，Spring 提供的 JMS 工具类，用它发送、接收消息。 -->
    <bean id="jmsTemplateQueue" class="org.springframework.jms.core.JmsTemplate">
        <property name="connectionFactory" ref="cachingConnectionFactory"/>
        <property name="defaultDestination" ref="defaultDestinationQueue"/>
        <!-- 非pub/sub模型（发布/订阅），true为topic,false为queue --> 
        <property name="pubSubDomain" value="false"/>
    </bean>
    
    <!-- 配置主题 JMS 模板，Spring 提供的 JMS 工具类，用它发送、接收消息。 -->
    <bean id="jmsTemplateTopic" class="org.springframework.jms.core.JmsTemplate">
        <property name="connectionFactory" ref="cachingConnectionFactory"/>
        <property name="defaultDestination" ref="defaultDestinationTopic"/>
        <!-- 非pub/sub模型（发布/订阅），true为topic,false为queue --> 
        <property name="pubSubDomain" value="true"/>
    </bean>
    
    <!--配置监听容器-->
    <bean class="org.springframework.jms.listener.DefaultMessageListenerContainer">
        <!-- 连接池 -->
        <property name="connectionFactory" ref="cachingConnectionFactory" />
        <!-- 监听目的地，这里我们配置队列的，如果想监听主题，换成主题目的地就可以了 -->
        <property name="destination" ref="defaultDestinationQueue" />
        <!-- 监听对象 -->
        <property name="messageListener" ref="queueConsumer" />
    </bean>
    
    <!-- 消息生产者 -->
    <bean id="queueProducer" class="com.example.activemq.QueueProducer">
        <property name="jmsTemplate" ref="jmsTemplateQueue"/>
    </bean>
    
    <!-- 消息消费者 -->
    <bean id="queueConsumer" class="com.example.activemq.QueueConsumer" />
</beans>
```

最后的项目结构如下：

![image](https://miansen.wang/assets/20200513162751.png)

为了演示简单，我直接在 main 方法里测试了。

```java
public class ApplicationTest {

    @SuppressWarnings("resource")
    public static void main(String[] args) {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("application-context.xml");
        QueueProducer producer = context.getBean(QueueProducer.class);
        Map<String, Object> map = new HashMap<>();
        map.put("name", "Hello ActiveMQ");
        producer.sendMessage(map);
    }

}
```

运行 main 方法后，就能看到消费者消费了消息。打开控制台页面，也可以看到消费的信息。

![image](https://miansen.wang/assets/20200513163048.png)

上面只演示了队列模式，主题模式也是一样的，就不一一演示了。