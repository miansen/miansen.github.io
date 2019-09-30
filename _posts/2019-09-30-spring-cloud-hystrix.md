---
layout: post
title: 五、SpringCloud-服务容错 Hystrix（断路器）
date: 2019-09-30
categories: SpringCloud
tags: SpringCloud Hystrix
author: 龙德
---

* content
{:toc}

在微服务架构中，服务与服务之间相互调用，互相依赖。如果其中的一个服务发故障，有可能导致整个系统不能用。Hystrix 就是用来预防这种情况的。

## 在 Ribbon 中使用 Hystrix

把 `spring-cloud-consumer-ribbon` 工程复制一份，命名为 `spring-cloud-consumer-ribbon-hystrix`

引入 `Hystrix` 依赖

```xml
<!-- 引入 Hystrix 依赖-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

`application.properties` 改一下服务名字就可以了

```properties
#指定服务名称
spring.application.name=spring-cloud-consumer-ribbon-hystrix
#指定运行端口
server.port=8077
#获取注册实例列表
eureka.client.fetch-registry=true
#注册到 Eureka 的注册中心
eureka.client.register-with-eureka=true
#配置注册中心地址
eureka.client.service-url.defaultZone=http://localhost:8080/eureka
```

启动类改成 `ConsumerRibbonHystrixApplication`，并且加上注解 `@EnableCircuitBreaker`，开启断路器 `Hystrix`

为使示例简单，`controller` 是写在启动类里的。所以我们在启动类里新建一个 `getUserFallback()` 方法。

```java
@GetMapping("/users/{name}")
@HystrixCommand(fallbackMethod = "getUserFallback")
public String getUser(@PathVariable("name") String name) {
	return restTemplate.getForObject("http://spring-cloud-provider/users/" + name, String.class);
}

public String getUserFallback(String name) {
	return "get " + name +" error";
}
```

`getUser()` 方法多了 `@HystrixCommand` 注解，它的作用是指定 `Hystrix` 在此方法超时时调用的方法。

可以在配置文件里指定超时时间

```properties
#自定义 Hystrix 的超时时间（毫秒）
hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds=3000
```

### 测试

写好之后测试一下

启动注册中心 `spring-cloud-eureka-server` 和服务提供者 `spring-cloud-provider`，紧接着启动现在的服务 `spring-cloud-consumber-ribbon-hystrix`

启动成功后访问 `http://localhost:8077/users/zhangsan`

![image](https://miansen.wang/assets/20190930140256.png)

可以看到一切正常

接着把服务提供者 `spring-cloud-provider` 停掉，然后再访问。如果没有引入 `Hystrix` 的话，我们的访问应该会报错。

但是引入 `Hystrix` 后，服务就有了容错机制，出现错误后会走到我们指定的方法。

![image](https://miansen.wang/assets/20190930140913.png)

## 在 Feign 中使用 Hystrix

把 `spring-cloud-consumer-feign` 工程复制一份，命名为 `spring-cloud-consumer-feign-hystrix`

`Feign` 是自带断路器的，不过需要在配置文件中配置打开它

```properties
#指定服务名称
spring.application.name=spring-cloud-consumer-feign-hystrix
#指定运行端口
server.port=8076
#获取注册实例列表
eureka.client.fetch-registry=true
#注册到 Eureka 的注册中心
eureka.client.register-with-eureka=true
#配置注册中心地址
eureka.client.service-url.defaultZone=http://localhost:8080/eureka
#开启 Hystrix
feign.hystrix.enabled=true
```

注解 `@FeignClient` 加上 `fallback` 参数，指定由哪个类来处理错误。

```java
@FeignClient(name = "spring-cloud-provider", configuration = FeignContractConfig.class, fallback = UserFeignClientHystrixFallback.class)
public interface UserFeignClient {
```

`UserFeignClientHystrixFallback` 类要实现 `UserFeignClient` 接口，并注入到 `IOC` 容器中。

```java
@Component
public class UserFeignClientHystrixFallback implements UserFeignClient{

	@Override
	public String getUser(String name) {
		return "get " + name +" error";
	}

}
```

### 测试

启动 `spring-cloud-consumer-feign-hystrix` 工程，访问 `http://localhost:8076/users/zhangsan`，结果如图：

![image](https://miansen.wang/assets/20190930155226.png)

### 捕获异常

虽然我们已经解决了服务挂掉的问题，但是还不知道服务是咋挂掉的，要是能捕获异常就好了。

这点 `Hystrix` 也是支持的，看看下面的代码。

注解 `@FeignClient` 需要改变一下

```java
@FeignClient(name = "spring-cloud-provider", configuration = FeignContractConfig.class, fallbackFactory = UserFeignClientHystrixFallbackFactory.class)
```

这次 `fallbackFactory` 属性指定的是 `UserFeignClientHystrixFallbackFactory`

`UserFeignClientHystrixFallbackFactory` 类要实现 `FallbackFactory` 接口并注入 `IOC` 容器。

```java
@Component
public class UserFeignClientHystrixFallbackFactory implements FallbackFactory<UserFeignClient> {

	private static Logger log = LoggerFactory.getLogger(UserFeignClientHystrixFallbackFactory.class);
	
	@Override
	public UserFeignClient create(Throwable cause) {
		log.info("the server error is: {}", cause.getMessage());
		return new UserFeignClient() {
			@Override
			public String getUser(String name) {
				return "get " + name +" error";
			}
		};
	}
}
```

在 `create()` 这个工厂方法中，它的入参就是服务提供者的异常。

源码下载：[https://github.com/miansen/SpringCloud-Learn](https://github.com/miansen/SpringCloud-Learn)