---
layout: post
title: 三、SpringCloud-服务消费者（RestTemplate+Ribbon 与 Feign）
date: 2019-09-25
categories: SpringCloud
tags: SpringCloud
author: 龙德
---

* content
{:toc}

在上一篇 [SpringCloud-服务的注册与发现 Eureka](https://miansen.wang/2019/09/24/spring-cloud-eureka-server)，我们搭建了服务的注册中心 Eureka，负责各个服务的注册与发现。但是还遗留了一个问题，那就是服务之间的调用方式还没解决。这一篇讲一下服务调用的两种方式，第一种是 `RestTemplate + Ribbon`，第二种是 `Feign`。

## RestTemplate + Ribbon

为了直观一点，我们就不在原来的工程上面改造了。

我们新建一个 `spring-cloud-consumer-ribbon` 工程，它的 `pom.xml` 跟 `spring-cloud-consumer` 工程一样。

现在整个工程的结构如下：

![image](https://miansen.wang/assets/20190925131618.png)

修改 `application.properties`

```
#指定服务名称
spring.application.name=spring-cloud-consumer-ribbon
#指定运行端口
server.port=8077
#获取注册实例列表
eureka.client.fetch-registry=true
#注册到 Eureka 的注册中心
eureka.client.register-with-eureka=true
#配置注册中心地址
eureka.client.service-url.defaultZone=http://localhost:8080/eureka
```

修改启动类

将 `RestTemplate` 注入到 IOC 容器中，并添加 `@LoadBalanced` 注解，开启负载均衡。

```java
@EnableEurekaClient
@RestController
@SpringBootApplication
public class ConsumerRibbonApplication {

	public static void main(String[] args) {
		SpringApplication.run(ConsumerRibbonApplication.class, args);
	}
	
	@Bean
	@LoadBalanced
	public RestTemplate restTemplate() {
		return new RestTemplate();
	}
	
	@Autowired
	private RestTemplate restTemplate;
	
	@GetMapping("/info")
	public String info() {
		return restTemplate.getForObject("http://spring-cloud-provider/info", String.class);
	}

}
```

简单介绍一下 RestTemplate 和 Ribbon
> RestTemplate 是 Spring 提供的用于访问 Rest 服务的客户端，RestTemplate 提供了多种便捷访问远程 Http 服务的方法，能够大大提高客户端的编写效率。
>
> Spring Cloud Ribbon 是一个基于 Http 和 TCP 的客服端负载均衡工具，它是基于 Netflix Ribbon 实现的。与 Eureka 配合使用时，Ribbon 可自动从 Eureka Server (注册中心)获取服务提供者地址列表，并基于负载均衡算法，通过在客户端中配置 ribbonServerList 来设置服务端列表去轮询访问以达到均衡负载的作用。

接着我们启动服务注册中心（spring-cloud-eureka-server）、服务提供者（多个）和服务消费者（spring-cloud-consumer-ribbon）

![image](https://miansen.wang/assets/20190925153551.png)

可以看到启动了一个 `spring-cloud-provider` 服务，它有3个实例，端口分别是 `8078`、`8081`、`8082`。

还有一个 `spring-cloud-consumer-ribbon` 服务，端口是 `8077`。

我们访问 `spring-cloud-consumer-ribbon` 服务的地址 `http://localhost:8077/info`，每次访问都会轮流的调用不同端口的服务。

![image](https://miansen.wang/assets/ribbon.gif)

### 自定义负载均衡策略

从上面例子我们知道 `Ribbon` 默认的负载均衡策略是才用轮训的方式，我们也可以自定义负载均衡的策略。

在启动类的上级新建包 `config`，然后新建 `LoadBalancedConfig` 类。

```java
@Configuration
public class LoadBalancedConfig {

	@Bean
	public IRule ribbonRule() {
		// return new RoundRobinRule(); // 轮训
		// return new WeightedResponseTimeRule(); // 加权权重
		// return new RetryRule(); // 带有重试机制的轮训
		// return new RandomRule(); // 随机
		return new MyRule(); // 自定义规则
	}
}
```

MyRule 类如下：

```java
public class MyRule extends AbstractLoadBalancerRule {

	@Override
	public Server choose(Object key) {
		List<Server> servers = getLoadBalancer().getAllServers();
		// 取服务列表的第一个
		return servers.get(0);
	}

	@Override
	public void initWithNiwsConfig(IClientConfig clientConfig) {
		// TODO Auto-generated method stub
		
	}
}
```

在启动类上添加注解 `@RibbonClient`，指定 `spring-cloud-provider` 服务使用 `LoadBalancedConfig` 提供的规则。

```java
@RibbonClient(name = "spring-cloud-provider", configuration = org.spring.cloud.config.LoadBalancedConfig.class)
```

然后重新启动 `spring-cloud-consumer-ribbon` 服务，访问 `http://localhost:8077/info`，可以看到每次访问都是同一个端口。当然这个只是测试用例，现实中并不会采用这种负载均衡的策略。

![image](https://miansen.wang/assets/ribbon2.gif)

## Feign

先看一下 SpringCloud 官网对它的定义

> Feign 是一个声明式的 Web 服务客户端。它支持 Feign 本身的注解、JAX-RS 注解以及 SpringMVC 的注解。
> SpringCloud 集成 Ribbon 和 Eureka 以在使用 Feign 时提供负载均衡的http客户端。

新建一个 `spring-cloud-consumer-feign` 工程

引入 `Feign` 依赖

```xml
<!-- 引入 Feign 依赖-->
<dependency>
  	<groupId>org.springframework.cloud</groupId>
  	<artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

修改 `application.properties`

```
#指定服务名称
spring.application.name=spring-cloud-consumer-feign
#指定运行端口
server.port=8076
#获取注册实例列表
eureka.client.fetch-registry=true
#注册到 Eureka 的注册中心
eureka.client.register-with-eureka=true
#配置注册中心地址
eureka.client.service-url.defaultZone=http://localhost:8080/eureka
```

修改启动类，添加 `@EnableFeignClients` 注解，表明这是一个 Feign 客户端。

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

在启动类同级新建包 `client`，然后添加 `InfoFeignClient` 接口

```java
@FeignClient(name = "spring-cloud-provider")
public interface InfoFeignClient {

	@GetMapping("/info")
	public String getInfo();
	
}
```

注解 `@FeignClient(name = "spring-cloud-provider")` 指定了需要调用服务的名字

`@GetMapping` 就是 SpringMVC 的注解了。

我们在新建一个 `controller` 包，新建一个 `InfoController` 类

```java
@RestController
public class InfoController {

	@Autowired
	private InfoFeignClient infoFeignClient;
	
	@GetMapping("/info")
	public String getInfo() {
		return infoFeignClient.getInfo();
	}
}
```

然后启动 `spring-cloud-consumer-feign` 服务，访问 [http://localhost:8076/info](http://localhost:8076/info)，也能取得数据。