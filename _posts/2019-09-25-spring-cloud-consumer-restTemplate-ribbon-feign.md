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

为了直观一点，就不在原来的工程上面改造了。

（1）新建一个 `spring-cloud-consumer-ribbon` 工程，它的 `pom.xml` 跟 `spring-cloud-consumer` 工程一样。

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

（2）修改启动类

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
	
	@GetMapping("/users/{name}")
	public String getUser(@PathVariable("name") String name) {
		return restTemplate.getForObject("http://spring-cloud-provider/users/" + name, String.class);
	}

}
```

简单介绍一下 RestTemplate 和 Ribbon
> RestTemplate 是 Spring 提供的用于访问 Rest 服务的客户端，RestTemplate 提供了多种便捷访问远程 Http 服务的方法，能够大大提高客户端的编写效率。
>
> Spring Cloud Ribbon 是一个基于 Http 和 TCP 的客服端负载均衡工具，它是基于 Netflix Ribbon 实现的。与 Eureka 配合使用时，Ribbon 可自动从 Eureka Server (注册中心)获取服务提供者地址列表，并基于负载均衡算法，通过在客户端中配置 ribbonServerList 来设置服务端列表去轮询访问以达到均衡负载的作用。

（3）启动服务注册中心（spring-cloud-eureka-server）、服务提供者（多个）和服务消费者（spring-cloud-consumer-ribbon）

![image](https://miansen.wang/assets/20190925153551.png)

可以看到启动了一个 `spring-cloud-provider` 服务，它有3个实例，端口分别是 `8078`、`8081`、`8082`。

还有一个 `spring-cloud-consumer-ribbon` 服务，端口是 `8077`。

访问 `spring-cloud-consumer-ribbon` 服务 `http://localhost:8077/users/zhangsan `，每次访问都会轮流的调用不同端口的消费提供者。

![image](https://miansen.wang/assets/ribbon.gif)

### 自定义负载均衡策略

从上面例子我们知道 `Ribbon` 默认的负载均衡策略是采用轮训的方式，我们也可以自定义负载均衡的策略。

（1）在启动类的上级新建包 `config`，然后新建 `LoadBalancedConfig` 类。

注意避免 SpringBoot 的包扫描，因为自定义的策略必须在 Eureka 的策略实例化以后再实例化才会生效。

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

项目结构如下

![image](https://miansen.wang/assets/20190926151929.png)

（2）在启动类上添加注解 `@RibbonClient`，指定 `spring-cloud-provider` 服务使用 `LoadBalancedConfig` 提供的规则。

```java
@RibbonClient(name = "spring-cloud-provider", configuration = org.spring.cloud.config.LoadBalancedConfig.class)
```

（3）然后重新启动 `spring-cloud-consumer-ribbon` 服务，访问 `http://localhost:8077/users/zhangsan`，可以看到每次访问都是同一个端口。当然这个只是测试用例，现实中并不会采用这种负载均衡的策略。

![image](https://miansen.wang/assets/ribbon2.gif)

## Feign

先看一下 SpringCloud 官网对它的定义

> Feign 是一个声明式的 Web 服务客户端。它支持 Feign 本身的注解、JAX-RS 注解以及 SpringMVC 的注解。
> SpringCloud 集成 Ribbon 和 Eureka 以在使用 Feign 时提供负载均衡的http客户端。

（1）新建一个 `spring-cloud-consumer-feign` 工程

引入 `Feign` 依赖

```xml
<!-- 引入 Feign 依赖-->
<dependency>
  	<groupId>org.springframework.cloud</groupId>
  	<artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

（2）修改 `application.properties`

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

（3）修改启动类，添加 `@EnableFeignClients` 注解，表明这是一个 Feign 客户端。

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

（4）在启动类同级新建包 `client`，然后添加 `UserFeignClient` 接口

```java
@FeignClient(name = "spring-cloud-provider")
public interface UserFeignClient {

	@GetMapping("/users/{name}")
	public String getUser(@PathVariable("name") String name);
	
}
```

注解 `@FeignClient(name = "spring-cloud-provider")` 指定了调用哪个服务。

`@GetMapping` 就是 SpringMVC 的注解了。

（5）新建一个 `controller` 包，新建一个 `UserController` 类

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

项目结构如下

![image](https://miansen.wang/assets/20190926153306.png)

（6）启动 `spring-cloud-consumer-feign` 服务，访问 `http://localhost:8076/users/zhangsan`，同样能取得数据。

![image](https://miansen.wang/assets/20190926153418.png)

### 修改 Feign 的默认配置

修改 Feign 的默认配置也存在包扫描的问题，跟修改 Ribbon 的策略一样，我们把包放在 SpringBoot 扫描不到的地方。

（1）新建一个 `config` 包，新建类 `FeignContract`

```java
@Configuration
public class FeignContractConfig {

	@Bean
	public Contract feignContract() {
		return new Contract.Default();
	}
}
```

项目结构如下

![image](https://miansen.wang/assets/20190926155113.png)

（2）在 `UserFeignClient` 类中的注解 `@FeignClient` 指定 `configuration` 参数。

`@FeignClient(name = "spring-cloud-provider", configuration = FeignContractConfig.class)`

我们在 `FeignContractConfig` 类中修改了 `Feign` 的 `Contract` ，`Contract` 是一个契约的概念。 `Feign` 默认的契约是 `SpringMVC`，所以我们在 `UserFeignClient` 类中使用的是 `SpringMVC` 的注解。现在 `Contract.Default()` 使用的契约是 `Feign `自己的，也就是说我们要把 `SpringMVC` 的注解修改为 `Feign` 的注解，否则项目启动不了。

```java
// @FeignClient("spring-cloud-provider")
@FeignClient(name = "spring-cloud-provider", configuration = FeignContractConfig.class)
public interface UserFeignClient {
	
	// SpringMVC 版本
	
	/*@GetMapping("/users/{name}")
	public String getUser(@PathVariable("name") String name) {
		return userFeignClient.getUser(name);
	}*/
	
	// Feign 版本
	@RequestLine("GET /users/{name}")
	public String getUser(@Param ("name") String name);
	
}
```

源码下载[https://github.com/miansen/SpringCloud-Learn](https://github.com/miansen/SpringCloud-Learn)