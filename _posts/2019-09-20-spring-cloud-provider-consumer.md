---
layout: post
title: 一、SpringCloud-服务提供者与服务消费者
date: 2019-09-20
categories: SpringCloud
tags: SpringCloud
author: 龙德
---

* content
{:toc}

本篇以及后面的所有文章都是使用如下环境：

- JDK 版本：1.8

- IDA：Eclipse

- Maven 版本：3.5.0

- SpringBoot 版本：2.1.8.RELEASE

- SpringCloud 版本：Greenwich.RELEASE

（1）首先创建一个 Maven 工程作为父工程，所有的子工程都继承这个父工程，用以管理子工程的版本依赖。

父工程的 `pom.xml` 如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	
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

父工程只留一个 `pom.xml` 文件就可以了，其他的文件目录都可以不要

（2）然后在父工程下新建一个服务提供者工程 spring-cloud-provider

`pom.xml` 如下：

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

（3）在 `resources` 文件夹下新建 `application.properties`

添加如下内容：

```
# 服务名字
spring.application.name=spring-cloud-provider
# 在 8078 端口启动
server.port=8078
```

（4）在启动类里简单的输出一些信息

```java
@RestController
@SpringBootApplication
public class ProviderApplication {

	public static void main(String[] args) {
		SpringApplication.run(ProviderApplication.class, args);
	}

	@Value("${server.port}")
    private String port;
	
	@GetMapping("/users/{name}")
	public String getUser(@PathVariable("name") String name) {
		return "user name: "+ name +", server name: spring-cloud-provider, port: " + port;
	}
}
```

（5）在父工程下新建一个服务消费者工程 spring-cloud-consumer

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

（6）在 `resources` 文件夹下新建 `application.properties`

添加如下内容：

```
# 服务名字
spring.application.name=spring-cloud-consumer
# 在 8079 端口启动
server.port=8079
```

（7）在服务消费者的启动类里调用服务提供者

```java
@RestController
@SpringBootApplication
public class ConsumerApplication {

	public static void main(String[] args) {
		SpringApplication.run(ConsumerApplication.class, args);
	}
	
	@GetMapping("/users/{name}")
	public String getUser(@PathVariable("name") String name) {
		return new RestTemplate().getForObject("http://localhost:8078/users/" + name, String.class);
	}

}
```

服务消费者的 getUser() 方法没有自己实现，而是调用的服务提供者的 getUser() 方法。

（8）创建好的整体工程结构如下

![image](https://miansen.wang/assets/20190920171238.png)

（9）依次启动服务提供者和服务消费者

（10）访问服务消费者的地址 `http://localhost:8079/users/zhangsan`，同样也能取得服务提供者的信息。

![image](https://miansen.wang/assets/20190926141752.png)

这样一个简单的伪 `SpringCloud` 项目的服务提供者和消费者就已经完成了。

源码下载[https://github.com/miansen/SpringCloud-Learn](https://github.com/miansen/SpringCloud-Learn)