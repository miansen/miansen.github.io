---
layout: post
title: 二、Spring Cloud-服务注册中心 Eureka
date: 2019-09-24
categories: SpringCloud
tags: SpringCloud
author: 龙德
---

* content
{:toc}

在上一篇 [SpringCloud-服务提供者与服务消费者](https://miansen.wang/2019/09/20/spring-cloud-provider-consumer) 中，服务提供者和服务消费者其实是两个独立的应用，并且服务调用的地址是硬编码在代码中的。我们需要一个注册中心（Eureka）来调度各个服务，并且监控各个服务的健康状态。

> Spring Cloud Eureka 是 Spring Cloud Netflix 微服务套件的一部分，基于 Netflix Eureka 做了二次封装，主要负责实现微服务架构中的服务治理功能。

## 创建服务注册中心

新建一个子工程 spring-cloud-eureka-server，作为服务注册中心。

**pom.xml**

```xml
<?xml version="1.0"?>
<project
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd"
	xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
	<modelVersion>4.0.0</modelVersion>

	<parent>
		<groupId>com.example</groupId>
		<artifactId>spring-cloud-pom</artifactId>
		<version>0.0.1-SNAPSHOT</version>
	</parent>

	<artifactId>spring-cloud-eureka-server</artifactId>
	<name>spring-cloud-eureka-server</name>
	<url>http://maven.apache.org</url>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
	</properties>

	<dependencies>
		<!-- eureka server -->
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
		</dependency>
	</dependencies>
</project>
```

**application.properties**

```properties
# 服务名称
spring.application.name=spring-cloud-eureka-server
# 运行端口
server.port=8070
# 是否要从注册中心获取服务（注册中心不需要开启）
eureka.client.fetch-registry=false
# 是否要注册到注册中心（注册中心不需要开启）
eureka.client.register-with-eureka=false
# 关闭保护模式
eureka.server.enable-self-preservation=false
```

**启动类**

```java
@EnableEurekaServer
@SpringBootApplication
public class EurekaServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaServerApplication.class, args);
	}
}
```

启动类多了一个 @EnableEurekaServer 注解，表示开启 Eureka Server。

启动工程，访问 [http://localhost:8070](http://localhost:8070)，便会看到 Eureka 提供的 Web 控制台。

界面如下：

![image](https://miansen.wang/assets/20190924165955.png)

因为没有注册服务所以提示 "No instances available"。

## 创建服务提供者

改造原先的 spring-cloud-provider 工程

**pom.xml 添加 Eureka Client 依赖**

```xml
<!-- 引入 eureka client 依赖-->
<dependency>
   <groupId>org.springframework.cloud</groupId>
   <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

**修改 application.properties 文件，添加以下配置：**

```properties
# 指定服务名称
spring.application.name=spring-cloud-provider
# 指定运行端口
server.port=8071
# 获取注册实例列表
eureka.client.fetch-registry=true
# 注册到 Eureka 的注册中心
eureka.client.register-with-eureka=true
# 配置注册中心地址
eureka.client.service-url.defaultZone=http://localhost:8070/eureka
```

**启动类**

```java
@EnableEurekaClient
@SpringBootApplication
public class ProviderApplication {

	public static void main(String[] args) {
		SpringApplication.run(ProviderApplication.class, args);
	}
	
}
```

启动类多了一个 @EnableEurekaClient 注解，表示将此服务注册到 Eureka Server。

接着启动 spring-cloud-provider 工程，刷新 [http://localhost:8070](http://localhost:8070)，可以看到 spring-cloud-provider 服务注册到 Eureka 了。

![image](/assets/20191107153922.png)

一个服务可以有多个实例，我们修改 spring-cloud-provider 工程的 application.properties 文件，只需要修改端口号就行了。

比如我修改成 8072，然后再启动。（如果你是用 IDEA，需要点击 Edit Configuration，将默认的 Single instance only (单实例) 的钩去掉才能启动）

启动成功后刷新 [http://localhost:8070](http://localhost:8070)，可以看到 spring-cloud-provider 服务有两个实例（相当于一个小集群）。

![image](/assets/20191107154152.png)

## 创建服务消费者

改造原先的 spring-cloud-consumer 工程

**pom.xml 添加 Eureka Client 依赖**

```xml
<!-- 引入 eureka client 依赖-->
<dependency>
   <groupId>org.springframework.cloud</groupId>
   <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

**修改 application.properties 文件，添加以下配置：**

```properties
# 指定服务名称
spring.application.name=spring-cloud-consumer
# 指定运行端口
server.port=8073
# 获取注册实例列表
eureka.client.fetch-registry=true
# 注册到 Eureka 的注册中心
eureka.client.register-with-eureka=true
# 配置注册中心地址
eureka.client.service-url.defaultZone=http://localhost:8070/eureka
```

**启动类**

```java
@EnableEurekaClient
@SpringBootApplication
public class ConsumerApplication {

	public static void main(String[] args) {
		SpringApplication.run(ConsumerApplication.class, args);
	}
	
}
```

之前是通过服务接口的具体地址来调用的，既然用了注册中心，那么客户端调用的时候肯定是不需要关心有多少个服务提供接口，下面我们来改造之前的调用代码。

首先改造 RestTemplate 的配置，添加一个 @LoadBalanced 注解，这个注解会自动构造 LoadBalancerClient 接口的实现类并注册到 Spring 容器中，代码如下所示。

```java
@Configuration
public class BeanConfiguration {

	@Bean
	@LoadBalanced
	public RestTemplate getRestTemplate() {
		return new RestTemplate();
	}
}
```

接下来就是改造调用代码，我们不再直接写固定地址，而是写成服务的名称，这个名称就是我们注册到 Eureka 中的名称，是属性文件中的 spring.application.name，相关代码如下所示。

```java
@RestController
public class UserController {

	@Autowired
	private RestTemplate restTemplate;
	
	@GetMapping("/users/{name}")
	public String getUser(@PathVariable("name") String name) {
		return restTemplate.getForObject("http://spring-cloud-provider/users/" + name, String.class);
	}
}
```

在 Eureka 中，一个工程的 spring.application.name 的属性值对应着一个服务的 ID，Eureka 可以根据这个 ID 去调度服务，我们无需关心具体的 IP 和端口，只要知道服务提供者的名字就可以调用了。

启动 spring-cloud-consumer 工程，刷新 [http://localhost:8070](http://localhost:8070)，可以看到服务消费者也注册到 Eureka 了。

![image](/assets/20191107155429.png)

访问服务消费者 [http://localhost:8073/users/zhangsan](http://localhost:8073/users/zhangsan) ，如果返回 "hello zhangsan" 字符串，说明我们调用成功了。

源码下载：[https://github.com/miansen/SpringCloud-Learn](https://github.com/miansen/SpringCloud-Learn)