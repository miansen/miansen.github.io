---
layout: post
title: 二、SpringCloud-服务的注册与发现 Eureka
date: 2019-09-24
categories: SpringCloud
tags: SpringCloud
author: 龙德
---

* content
{:toc}

在上一篇 [SpringCloud-服务提供者与服务消费者](https://miansen.wang/2019/09/20/spring-cloud-provider-consumer) 中，服务提供者和服务消费者其实是两个独立的应用，并且服务调用的地址是硬编码在代码中的。我们需要一个注册中心来调度各个服务，并且监控各个服务的健康状态。

## 创建服务注册中心

（1）在父工程下新建一个子工程 `spring-cloud-eureka-server`

`pom.xml` 文件如下：

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
  
  <artifactId>spring-cloud-eureka-server</artifactId>
  <name>spring-cloud-eureka-server</name>
  <url>http://maven.apache.org</url>
  
  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>
  
  <dependencies>
    <!-- 引入 web 依赖 -->
  	<dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
   	</dependency>
    <!-- 引入 eureka server 依赖-->
	<dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
  </dependencies>
</project>
```

（2）在 `resources` 文件夹下新建 `application.properties`

添加如下内容：

```
#指定服务名称
spring.application.name=spring-cloud-eureka-server
#指定运行端口
server.port=8080
#指定主机地址
eureka.instance.hostname=localhost
#指定是否要从注册中心获取服务（注册中心不需要开启）
eureka.client.fetch-registry=false
#指定是否要注册到注册中心（注册中心不需要开启）
eureka.client.register-with-eureka=false
#关闭保护模式
eureka.server.enable-self-preservation=false
```

（3）启动类添加 `@EnableEurekaServer` 注解

```java
@EnableEurekaServer
@RestController
@SpringBootApplication
public class EurekaServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaServerApplication.class, args);
	}
}
```

（4）启动工程，访问 [http://localhost:8080](http://localhost:8080)

界面如下：

![image](https://miansen.wang/assets/20190924165955.png)

因为没有注册服务所以提示 `No instances available`

## 创建服务提供者

（1）在原先的 `spring-cloud-provider` 工程中添加 `Eureka` 依赖

```xml
<!-- 引入 eureka server 依赖-->
<dependency>
   <groupId>org.springframework.cloud</groupId>
   <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

（2）修改 `application.properties` 文件，添加以下配置：

```
#指定服务名称
spring.application.name=spring-cloud-provider
#指定运行端口
server.port=8078
#获取注册实例列表
eureka.client.fetch-registry=true
#注册到 Eureka 的注册中心
eureka.client.register-with-eureka=true
#配置注册中心地址
eureka.client.service-url.defaultZone=http://localhost:8080/eureka
```

（3）启动类添加 `@EnableEurekaClient` 注解，然后启动此工程。

刷新 [http://localhost:8080](http://localhost:8080)，可以看到 `spring-cloud-provider` 服务注册到 `Eureka` 了。

![image](https://miansen.wang/assets/20190924172714.png)

一个服务可以有多个实例，我们修改 `spring-cloud-provider` 工程的 `application.properties` 文件，只需要修改端口号就行了。

比如我修改成 `8081`，然后再启动。（如果你是用 IDEA，需要点击 Edit Configuration，将默认的 Single instance only (单实例) 的钩去掉才能启动）

启动成功后刷新 [http://localhost:8080](http://localhost:8080)，可以看到 `spring-cloud-provider` 服务有两个实例（相当于一个小集群）。

![image](https://miansen.wang/assets/20190924173500.png)

同理我们也可以注册多个服务。

修改 `spring-cloud-provider` 工程的 `application.properties` 文件，

将服务名称改为 `spring-cloud-provider-2`，端口号改为 `8082`，然后再启动。

启动成功后刷新 [http://localhost:8080](http://localhost:8080)，可以看到服务注册中心注册了两个服务，一个是 `spring-cloud-provider`，另一个是 `spring-cloud-provider-2`,其中 `spring-cloud-provider` 服务有两个实例。

![image](https://miansen.wang/assets/20190924173847.png)

## 创建服务消费者

（1）在原先的 `spring-cloud-consumer` 工程中添加 `Eureka` 依赖

```xml
<!-- 引入 eureka server 依赖-->
<dependency>
   <groupId>org.springframework.cloud</groupId>
   <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

（2）修改 `application.properties` 文件，添加以下配置：

```
#指定服务名称
spring.application.name=spring-cloud-consumer
#指定运行端口
server.port=8079
#获取注册实例列表
eureka.client.fetch-registry=true
#注册到 Eureka 的注册中心
eureka.client.register-with-eureka=true
#配置注册中心地址
eureka.client.service-url.defaultZone=http://localhost:8080/eureka
```

（3）修改启动类

服务消费者的启动类添加 `@EnableEurekaClient` 注解，并且将 `"http://localhost:8078/users/" + name` 替换成 `"http://spring-cloud-provider/users/" + name`

在 `Eureka` 中，一个工程的 `spring.application.name` 的属性值对应着一个服务的 ID，`Eureka` 可以根据这个 ID 去调度服务，我们无需关心具体的 IP + 端口，只要知道服务提供者的名字就可以调用了。

```java
@EnableEurekaClient
@RestController
@SpringBootApplication
public class ConsumerApplication {

	public static void main(String[] args) {
		SpringApplication.run(ConsumerApplication.class, args);
	}
	
	@GetMapping("/users/{name}")
	public String getUser(@PathVariable("name") String name) {
		return new RestTemplate().getForObject("http://spring-cloud-provider/users/" + name, String.class);
	}

}
```

（4）启动服务消费者，刷新 [http://localhost:8080](http://localhost:8080)，可以看到服务消费者也注册到 `Eureka` 了。

![image](https://miansen.wang/assets/20190924181713.png)

不过这时候访问服务消费者 [http://localhost:8079/users/zhangsan](http://localhost:8079/users/zhangsan) 会报错，因为需要开启 `Eureka` 的负载均衡后才能调用服务提供者，下一篇会具体讲 Spring Cloud 服务之间的调用方式。

源码下载[https://github.com/miansen/SpringCloud-Learn](https://github.com/miansen/SpringCloud-Learn)