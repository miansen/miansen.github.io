---
layout: post
title: 一、Spring Cloud-服务提供者与服务消费者
date: 2019-09-20
categories: SpringCloud
tags: SpringCloud
author: 龙德
---

* content
{:toc}

本篇以及后面的系列文章都是使用以下环境：

- JDK 版本：1.8

- IDA：Eclipse 4.6.0

- Maven 版本：3.5.0

- SpringBoot 版本：2.1.8.RELEASE

- SpringCloud 版本：Greenwich.RELEASE

## 父工程

创建一个父工程，命名为 spring-cloud-pom， 所有的子工程都继承这个父工程，用以管理子工程的版本依赖。

**pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	
	<!-- Spring Boot -->
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.1.8.RELEASE</version>
		<relativePath/>
	</parent>
	
	<groupId>com.example</groupId>
	<artifactId>spring-cloud-pom</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>pom</packaging>
	<name>spring-cloud-pom</name>

	<properties>
		<java.version>1.8</java.version>
		<maven-jar-plugin.version>3.1.1</maven-jar-plugin.version>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<spring-cloud.version>Greenwich.RELEASE</spring-cloud.version>
	</properties>
	
	<!-- Spring Cloud -->
	<dependencyManagement>
	   <dependencies>
	       <dependency>
	           <groupId>org.springframework.cloud</groupId>
	           <artifactId>spring-cloud-dependencies</artifactId>
	           <version>${spring-cloud.version}</version>
	           <type>pom</type>
	           <scope>import</scope>
	       </dependency>
	   </dependencies>
	</dependencyManagement>

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

父工程只留一个 pom.xml 文件就可以了，其他的文件目录都可以不要。

## 服务提供者

在父工程下新建一个子工程 spring-cloud-provider，作为服务提供者。

**pom.xml**

```xml
<?xml version="1.0"?>
<project xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd" xmlns="http://maven.apache.org/POM/4.0.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <modelVersion>4.0.0</modelVersion>
  
  <parent>
    <groupId>com.example</groupId>
    <artifactId>spring-cloud-pom</artifactId>
    <version>0.0.1-SNAPSHOT</version>
  </parent>
  
  <artifactId>spring-cloud-provider</artifactId>
  <name>spring-cloud-provider</name>
  <url>http://maven.apache.org</url>
  
  <dependencies>
  	<!-- 引入 web 依赖 -->
  	<dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
   </dependency>
  </dependencies>
</project>
```

**application.properties**

```properties
# 服务名字
spring.application.name=spring-cloud-provider
# 在 8071 端口启动
server.port=8071
```

**controller**

创建一个 Controller，提供一个接口给其他服务查询，简单的输出一些信息即可。

```java
@RestController
public class UserController {

	@GetMapping("/users/{name}")
	public String users(@PathVariable("name") String name) {
		return "hello " + name;
	}
}
```

启动服务，访问 http://localhost:8071/user/zhangsan ，如果能看到我们返回的 "hello zhangsan" 字符串，就证明接口提供成功了。

## 服务消费者

在父工程下新建一个子工程 spring-cloud-consumer，作为服务消费者。

**pom.xml**

```xml
<?xml version="1.0"?>
<project xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd" xmlns="http://maven.apache.org/POM/4.0.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <modelVersion>4.0.0</modelVersion>
  
  <parent>
    <groupId>com.example</groupId>
    <artifactId>spring-cloud-pom</artifactId>
    <version>0.0.1-SNAPSHOT</version>
  </parent>
  
  <artifactId>spring-cloud-consumer</artifactId>
  <name>spring-cloud-consumer</name>
  <url>http://maven.apache.org</url>
  
  <dependencies>
  	<!-- 引入 web 依赖 -->
  	<dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
   </dependency>
  </dependencies>
</project>
```

**application.properties**

```properties
# 服务名字
spring.application.name=spring-cloud-consumer
# 在 8072 端口启动
server.port=8072
```

**RestTemplate**

RestTemplate 是 Spring 提供的用于访问 Rest 服务的客户端，RestTemplate 提供了多种便捷访问远程 Http 服务的方法，能够大大提高客户端的编写效率。我们通过配置 RestTemplate 来调用接口。

```java
@Configuration
public class BeanConfiguration {

	@Bean
	public RestTemplate getRestTemplate() {
		return new RestTemplate();
	}
}
```

**controller**

创建 controller，利用 RestTemplate 访问服务提供者的接口。

```java
@RestController
public class UserController {

	@Autowired
	private RestTemplate restTemplate;
	
	@GetMapping("/users/{name}")
	public String getUser(@PathVariable("name") String name) {
		return restTemplate.getForObject("http://localhost:8071/users/" + name, String.class);
	}
}
```

服务消费者的 getUser() 方法没有自己实现，而是调用服务提供者的接口。

启动服务消费者，访问 http://localhost:8072/users/zhangsan ，如果返回 "hello zhangsan" 字符串就证明调用成功了。

这样一个简单的伪微服务项目的服务提供者和消费者就已经完成了。

之所以说是伪微服务，是因为服务提供者和服务消费者其实是两个独立的应用，它们之间只是简单的通过 http 的方式进行资源访问和操作。而真正微服务架构不仅仅是将单体应用划分为小型的服务单元这么简单，还需要一套的基础组件来管理各个服务，如服务注册、服务发现、配置中心、消息总线、负载均衡、断路器、数据监控等。Spring Cloud 就是这一系列微服务基础组件框架的集合，利用 Spring Boot 的开发便利性，巧妙地简化了微服务系统基础设施的开发。

通俗地讲，Spring Cloud 就是用于构建微服务开发和治理的框架集合（它并不是具体的一个框架，而是框架集合）

Spring Cloud 提供的微服务基础组件有：

- Eureka：服务注册中心，用于服务管理
- Ribbon：基于客户端的负载均衡组件
- Hystrix：容错框架，能够防止服务的雪崩效应
- Feign：Web 服务客户端，能够简化 HTTP 接口的调用
- Zuul：API 网关，提供路由转发、请求过滤等功能
- Config：分布式配置管理
- Sleuth：服务跟踪
- Stream：构建消息驱动的微服务应用程序的框架
- Bus：消息代理的集群消息总线

所以接下来，我们利用 Spring Cloud 提供的组件，一步步的构建出一个真正的微服务系统（Demo）。

源码下载：[https://github.com/miansen/SpringCloud-Learn](https://github.com/miansen/SpringCloud-Learn)