---
layout: post
title: 四、Spring Cloud-接口调用 Feign
date: 2019-09-26
categories: Spring全家桶
tags: SpringCloud
author: 龙德
---

* content
{:toc}

先看一下 SpringCloud 官网对它的定义

> Feign 是一个声明式的 Web 服务客户端。它支持 Feign 本身的注解、JAX-RS 注解以及 SpringMVC 的注解。
> SpringCloud 集成 Ribbon 和 Eureka 以在使用 Feign 时提供负载均衡的 http 客户端。

## 在 Spring Cloud 中使用 Feign

新建一个子工程，命名为 spring-cloud-consumer-feign

引入 Feign 依赖

```xml
<!-- 引入 Feign 依赖-->
<dependency>
  	<groupId>org.springframework.cloud</groupId>
  	<artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

application.properties

```properties
#指定服务名称
spring.application.name=spring-cloud-consumer-feign
#指定运行端口
server.port=8074
#获取注册实例列表
eureka.client.fetch-registry=true
#注册到 Eureka 的注册中心
eureka.client.register-with-eureka=true
#配置注册中心地址
eureka.client.service-url.defaultZone=http://localhost:8080/eureka
```

启动类添加 @EnableFeignClients 注解，表明这是一个 Feign 客户端。

```java
@EnableFeignClients
@EnableEurekaClient
@RestController
@SpringBootApplication
public class ConsumerFeignApplication {

	public static void main(String[] args) {
		SpringApplication.run(ConsumerFeignApplication.class, args);
	}
}
``` 

新建一个 UserFeignClient 接口 

```java
@FeignClient(name = "spring-cloud-provider")
public interface UserFeignClient {

	@GetMapping("/users/{name}")
	public String getUser(@PathVariable("name") String name);
	
}
```

注解 @FeignClient(name = "spring-cloud-provider") 指定了调用哪个服务。

新建一个 UserController 类，并注入 UserFeignClient，就像调用本地方法一样调用远程接口。

```java
@RestController
public class UserController {

	@Autowired
	private UserFeignClient userFeignClient;
	
	@GetMapping("/users/{name}")
	public String getUser(@PathVariable("name") String name) {
		return userFeignClient.getUser(name);
	}
}
```

启动 spring-cloud-eureka-server、spring-cloud-provider 和 spring-cloud-consumer-feign，访问 [http://localhost:8074/users/zhangsan](http://localhost:8074/users/zhangsan) 同样能取得数据。

## 继承特性

从上面的例子可以看到，UserFeignClient 和 UserController 其实是声明与实现的关系，Feign 是通过接口的形式调用的，利用 Feign 的继承特性，可以把服务的接口单独抽出来，作为公共的依赖，以方便使用。

### 定义公共的 API 接口

新建一个子工程，命名为 spring-cloud-feign-api

因为要用到 MVC 的注解，所以引入 Feign 依赖，当然引入 web 依赖也可以。

```xml
<!-- 引入 Feign 依赖-->
<dependency>
  	<groupId>org.springframework.cloud</groupId>
  	<artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

定义一个接口，我这里以 api 开头，以便区分。

```java
public interface UserApiService {
	
	@GetMapping("/api/users/{name}")
	public String getApiUser(@PathVariable("name") String name);
}
```

在实际项目中，这个工程应该要打成 jar 包的形式供其他工程使用的，这里我就不成 jar 包了，直接引用。

### 服务提供者实现接口

spring-cloud-provider 工程引入 spring-cloud-feign-api

```xml
<dependency>
	<groupId>com.example</groupId>
	<artifactId>spring-cloud-feign-api</artifactId>
	<version>0.0.1-SNAPSHOT</version>
</dependency>
```

新建一个 Controller，实现 UserApiService 接口

```java
@RestController
public class UserApiController implements UserApiService {

	@Override
	public String getUserApi(String name) {
		return name + ", from user api";
	}

}
```

不同的是这里不需要在方法上面添加 @GetMapping 注解，这些注解在父接口中都有，不过在 Controller 上还是要添加 @RestController 注解。

### 服务消费者继承接口

spring-cloud-consumer-feign 同样引入 spring-cloud-feign-api

然后 UserFeignClient 继承 UserApiService

```java
@FeignClient(name = "spring-cloud-provider")
public interface UserFeignClient extends UserApiService {
	
	@GetMapping("/users/{name}")
	public String getUser(@PathVariable("name") String name);
	
}
```

controller

```java
@RestController
public class UserController {

	@Autowired
	private UserFeignClient userFeignClient;
	
	@GetMapping("/users/{name}")
	public String getUser(@PathVariable("name") String name) {
		return userFeignClient.getUser(name);
	}
	
	@GetMapping("/api/users/{name}")
	public String getApiUser(@PathVariable("name") String name) {
		return userFeignClient.getUserApi(name);
	}
}
```

访问结果

![image](/assets/20191111171356.png)

## 修改 Feign 的默认配置

修改 Feign 的默认配置也存在包扫描的问题，跟修改 Ribbon 的策略一样，我们使用注解的方式忽略扫描的类。

新建一个 config 包，新建类 FeignContract。

```java
@ExcludeFromComponentScan
@Configuration
public class FeignContractConfig {

	@Bean
	public Contract feignContract() {
		return new Contract.Default();
	}
}
```

在 UserFeignClient 类中的注解 @FeignClient 指定 configuration 参数。

```java
@FeignClient(name = "spring-cloud-provider", configuration = FeignContractConfig.class)
```

启动类指定包扫描忽略使用 @ExcludeFromComponentScan 注解的类

```java
//忽略使用  @ExcludeFromComponentScan 注解的类
@ComponentScan(excludeFilters = {@ComponentScan.Filter(type = FilterType.ANNOTATION, value = ExcludeFromComponentScan.class)})
@EnableFeignClients
@EnableEurekaClient
@RestController
@SpringBootApplication
public class ConsumerFeignApplication {
```

我们在 FeignContractConfig 类中修改了 Feign 的 Contract ，Contract 是一个契约的概念。 Feign 默认的契约是 SpringMVC，所以我们在 UserFeignClient 类中使用的是 SpringMVC 的注解。现在 Contract.Default() 使用的契约是 Feign 自己的，也就是说我们要把 SpringMVC 的注解修改为 Feign 的注解，否则项目启动不了。

```java
@FeignClient(name = "spring-cloud-provider", configuration = FeignContractConfig.class)
public interface UserFeignClient extends UserApiService {
	
	// SpringMVC 版本
	// @GetMapping("/users/{name}")
	// public String getUser(@PathVariable("name") String name);
	
	// Feign 版本
	@RequestLine("GET /users/{name}")
	public String getUser(@Param ("name") String name);
	
}
``` 

源码下载：[https://github.com/miansen/SpringCloud-Learn](https://github.com/miansen/SpringCloud-Learn)