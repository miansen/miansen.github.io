---
layout: post
title: 七、Spring Cloud-网关 Spring-Cloud-Gateway
date: 2019-10-21
categories: SpringCloud
tags: Spring-Cloud-Gateway
author: 龙德
---

* content
{:toc}

## 简介

Spring Cloud Gateway 是 Spring Cloud 的一个全新项目，该项目是基于 Spring 5.0，Spring Boot 2.0 和 Project Reactor 等技术开发的网关，它旨在为微服务架构提供一种简单有效的统一的 API 路由管理方式。

Spring Cloud Gateway 作为 Spring Cloud 生态系统中的网关，目标是替代 Netflix Zuul，其不仅提供统一的路由方式，并且基于 Filter 链的方式提供了网关基本的功能，例如：安全，监控/指标，和限流。

## 概念

- Route（路由）：Route 是网关的基础元素，由 ID、目标 URI、断言、过滤器组成。当请求到达网关时，由 Gateway Handler Mapping 通过断言进行路由匹配，当断言为真时，匹配到路由。
- Predicate（断言）：Predicate 是 Java 8 中提供的一个函数。允许开发人员匹配来自 HTTP 的请求，例如请求头或者请求参数。简单来说它就是匹配条件。
- Filter（过滤器）：Filter 是 Gateway 中的过滤器，可以在请求发出前后进行一些业务上的处理。

## 工作原理

![image](/assets/spring-cloud-gateway.png)

当客户端请求到达 Spring Cloud Gateway 后，Gateway Handler Mapping 会将其拦截，根据 predicates 确定请求与哪个路由匹配。如果匹配成功，则会将请求发送至 Gateway web handler。Gateway web handler 处理请求会经过一系列 "pre" 类型的过滤器，然后执行代理请求。执行完之后再经过一系列的 "post" 类型的过滤器，最后返回给客户端。

## 快速开始

新建一个子工程，命名为 spring-cloud-gateway

引入依赖

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

application.properties 配置路由

```properties
spring.application.name=spring-cloud-gateway
server.port=8074

############ 定义了一个 router（注意是数组的形式） ############
# 路由 ID，保持唯一
spring.cloud.gateway.routes[0].id=my-gateway
# 目标服务地址
spring.cloud.gateway.routes[0].uri=http://httpbin.org
# 路由条件
spring.cloud.gateway.routes[0].predicates[0]=Path=/get
```

上面这段配置的意思是，配置了一个 id 为 my-gateway 的路由规则，当访问地址为 `/get` 时会自动转发到 [http://httpbin.org/get](http://httpbin.org/get)

还可以通过代码的形式配置路由

```java
/**
 * 通过代码的形式配置路由
 * @param builder
 * @return
 */
@Bean
public RouteLocator myRoutes(RouteLocatorBuilder builder) {
	return builder.routes()
			.route(
					p -> p.path("/get").uri("http://httpbin.org")
					)
			.build();
}
```

application.propertise 配置路由和代码配置路由选择其中一个就好了，个人推荐 application.propertise 的形式配置。

启动服务，访问 [http://localhost:8074/get](http://localhost:8074/get)

应该会输出以下内容：

```json
{
  "args": {}, 
  "headers": {
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3", 
    "Accept-Encoding": "gzip, deflate, br", 
    "Accept-Language": "zh-CN,zh;q=0.9", 
    "Cache-Control": "max-age=0", 
    "Cookie": "Webstorm-2f8f75da=e0c5ee46-9276-490c-b32b-d5dc1483ca18; acw_tc=2760828015735472938194099e940a3c3ebc07316bcb1096abc6fefde61bf8; BD_UPN=12314353; H_PS_645EC=8749qnWwXCzugp%2FwPJDVeB7bqBisqx6VKFthj5OZOsWBAz1JPX2YkatsizA; BD_HOME=0", 
    "Forwarded": "proto=http;host=\"localhost:8074\";for=\"0:0:0:0:0:0:0:1:51881\"", 
    "Host": "httpbin.org", 
    "Sec-Fetch-Mode": "navigate", 
    "Sec-Fetch-Site": "none", 
    "Sec-Fetch-User": "?1", 
    "Upgrade-Insecure-Requests": "1", 
    "User-Agent": "Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/78.0.3904.97 Safari/537.36", 
    "X-Forwarded-Host": "localhost:8074"
  }, 
  "origin": "0:0:0:0:0:0:0:1, 183.14.135.71, ::1", 
  "url": "https://localhost:8074/get"
}
```

## 整合 Eureka

添加 Eureka Client 的依赖

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

配置基于 Eureka 的路由

```properties
spring.application.name=spring-cloud-gateway
server.port=8074

########### 配置注册中心 ###########
# 获取注册实例列表
eureka.client.fetch-registry=true
# 注册到 Eureka 的注册中心
eureka.client.register-with-eureka=true
# 配置注册中心地址
eureka.client.service-url.defaultZone=http://localhost:8070/eureka

############ 定义了一个基于 Eureka 的 router（注意是数组的形式） ############
# 路由 ID，保持唯一
spring.cloud.gateway.routes[0].id=my-gateway
# 目标服务地址
spring.cloud.gateway.routes[0].uri=lb://spring-cloud-provider
# 路由条件
spring.cloud.gateway.routes[0].predicates[0]=Path=/user-service/**
```

uri 以 `lb://` 开头（lb 代表从注册中心获取服务），后面接的就是你需要转发到的服务名称，这个服务名称必须跟 Eureka 中的对应，否则会找不到服务。

spring-cloud-provider 服务提供的接口如下：

```java
@RestController
@RequestMapping("/user-service")
public class UserController {

	@Value("${spring.application.name}")
	private String applicationName;
	
	@Value("${server.port}")
	private String post;
	
	@GetMapping("/users/{name}")
	public String users(@PathVariable("name") String name) {
		return String.format("hello %s，from server %s，post: %s", name, applicationName, post);
	}
}
```

启动 spring-cloud-eureka-server（注册中心）、spring-cloud-provider 和 spring-cloud-gateway

访问 [http://localhost:8074/user-service/users/zhangsan](http://localhost:8074/user-service/users/zhangsan)，输出如下：

![image](/assets/20191114150630.png)

### 配置默认路由

Spring Cloud Gateway 提供了类似于 Zuul 那种为所有服务转发的功能

配置如下：

```properties
spring.application.name=spring-cloud-gateway
server.port=8074

########### 配置注册中心 ###########
# 获取注册实例列表
eureka.client.fetch-registry=true
# 注册到 Eureka 的注册中心
eureka.client.register-with-eureka=true
# 配置注册中心地址
eureka.client.service-url.defaultZone=http://localhost:8070/eureka

# 配置默认路由
spring.cloud.gateway.discovery.locator.enabled=true
```

开启之后我们需要通过地址去访问服务了，格式如下：

```
http://网关地址/服务名称(大写)/**
```

例如：[http://localhost:8074/SPRING-CLOUD-PROVIDER/user-service/users/zhangsan](http://localhost:8074/SPRING-CLOUD-PROVIDER/user-service/users/zhangsan)

结果如图：

![image](/assets/20191114152229.png)

服务名称也可以配置成小写的格式，只需要增加一条配置即可：

```properties
# 配置服务名称小写
spring.cloud.gateway.discovery.locator.lowerCaseServiceId=true
```

## 路由断言工厂

官方提供了很多个常用的路由断言工厂，如图所示：

![image](https://miansen.wang/assets/spring-cloud-gateway-predicates.png)

**1. Path 路由断言工厂**

Path 路由断言工厂接收一个参数，根据 Path 定义好的规则来判断访问的 URI 是否匹配

固定的 Path

```properties
# spring.cloud.gateway.routes[0].predicates[0]=Path=/users/zhangsan
```

带有前缀的 Path

```properties
# spring.cloud.gateway.routes[0].predicates[0]=Path=/users/{segment}
```

使用通配符的 Path

```properties
# spring.cloud.gateway.routes[0].predicates[0]=Path=/users/**
```

**2. Query 路由断言工厂**

Query 路由断言工厂接收两个参数，一个必需的参数和一个可选的正则表达式

```properties
# spring.cloud.gateway.routes[0].predicates[0]=Query=foo, ba.
```

如果请求包含 foo 查询参数，则此路由将匹配。bar 和 baz 也会匹配，因为第二个参数是正则表达式（注意 ba 后面有个 .）

测试链接：

http://localhost:8074/users/zhangsan?foo=ba

http://localhost:8074/users/zhangsan?foo=bar

http://localhost:8074/users/zhangsan?foo=baz

**3. Method 路由断言工厂**

Method 路由断言工厂接收一个参数，即要匹配的 HTTP 方法。

```properties
# spring.cloud.gateway.routes[0].predicates[0]=Method=GET
```

**4. Header 路由断言工厂**

Header 路由断言工厂接收两个参数，分别是请求头名称和正则表达式。

```properties
# spring.cloud.gateway.routes[0].predicates[0]=Header=X-Request-Id, \d+
```

如果请求中带有请求头名为 x-request-id，其值与 \d+ 正则表达式匹配（值为一个或多个数字），则此路由匹配。

具体的可以看一下官方文档，写的很清楚。

[https://cloud.spring.io/spring-cloud-static/spring-cloud-gateway/2.0.0.RELEASE/single/spring-cloud-gateway.html](https://cloud.spring.io/spring-cloud-static/spring-cloud-gateway/2.0.0.RELEASE/single/spring-cloud-gateway.html)

### 自定义路由断言工厂

自定义路由断言工厂需要继承 AbstractRoutePredicateFactory 类，重写 apply 方法的逻辑。

在 apply 方法中可以通过 exchange.getRequest() 拿到 ServerHttpRequest 对象，从而可以获取到请求的参数、请求方式、请求头等信息。

apply 方法的参数是自定义的配置类，也就是静态内部类 Config，在使用的时候配置参数，就可以在 apply 方法中直接获取使用。

我们自己写一个 Query 路由断言工厂吧，名字就叫 MyQueryRoutePredicateFactory（命名需要以 RoutePredicateFactory 结尾）

代码如下：

```java
@Component
public class MyQueryRoutePredicateFactory extends AbstractRoutePredicateFactory<MyQueryRoutePredicateFactory.Config> {

	public MyQueryRoutePredicateFactory() {
		super(Config.class);
	}
	
	/**
	 * 返回有关 args 数量和快捷方式分析顺序的提示。
	 * <p>必须要重写这个方法，否则 Config 设置不了参数。
	 */
	@Override
	public List<String> shortcutFieldOrder() {
		return Arrays.asList("param", "regexp");
	}



	/**
	 * 自己实现 Query 路由断言工厂
	 * <p>在这个方法里写逻辑
	 */
	@Override
	public Predicate<ServerWebExchange> apply(Config config) {
		return exchange -> {
			System.out.println(config.toString());
			if (config.getRegexp() == null || "".equals(config.getRegexp())) {
				return exchange.getRequest().getQueryParams().containsKey(config.getParam());
			}
			List<String> values = exchange.getRequest().getQueryParams().get(config.getParam());
			if (values == null) {
				return false;
			}
			for (String value : values) {
				if (value != null && value.matches(config.getRegexp())) {
					return true;
				}
			}
			return false;
		};
	}

	/**
	 * Config 静态内部类用来保存配置信息
	 * @author miansen.wang
	 * @date 2019-11-14
	 */
	public static class Config {

		// 请求参数名
		private String param;

		// 请求参数值的正则
		private String regexp;

		public String getParam() {
			return param;
		}

		public void setParam(String param) {
			this.param = param;
		}

		public String getRegexp() {
			return regexp;
		}

		public void setRegexp(String regexp) {
			this.regexp = regexp;
		}

		@Override
		public String toString() {
			return "Config {param=" + param + ", regexp=" + regexp + "}";
		}
		
	}
}
```

在配置文件中使用

```properties
spring.cloud.gateway.routes[0].predicates[0]=MyQuery=foo, ba.
```

重启服务，在 apply 方法处断点，访问 [http://localhost:8074/user-service/users/zhangsan?foo=bar](http://localhost:8074/user-service/users/zhangsan?foo=bar) 进入到 apply 方法，说明我们自定义路由断言工厂起作用了。

## 过滤器工厂

Spring Cloud Gateway 根据作用范围划分为 GatewayFilter 和 GlobalFilter，二者区别如下：

- GatewayFilter：需要通过 spring.cloud.routes.filters 配置在具体路由下，只作用在当前路由上
- GlobalFilter：全局过滤器，不需要在配置文件中配置，作用在所有的路由上

官方提供了很多 GatewayFilter 和 GlobalFilter，可以在这里查看： [https://cloud.spring.io/spring-cloud-static/spring-cloud-gateway/2.0.0.RELEASE/single/spring-cloud-gateway.html](https://cloud.spring.io/spring-cloud-static/spring-cloud-gateway/2.0.0.RELEASE/single/spring-cloud-gateway.html)

### GatewayFilter

先看看官方提供的 GatewayFilter

![image](https://miansen.wang/assets/GatewayFilter.jpg)

下面我们写一个例子，这个例子需要做到在请求到达服务之前添加一个请求头，服务响应之后添加一个响应头。

由图可知，添加请求头的 GatewayFilter 为 AddRequestHeaderGatewayFilterFactory（约定写成 AddRequestHeader）

添加响应头的 GatewayFilter 为 AddResponseHeaderGatewayFilterFactory（约定写成 AddResponseHeader）

所以，配置文件可以这样配置

```properties
spring.application.name=spring-cloud-gateway
server.port=8074

########### 通过配置的形式配置路由 ###########
# 自定义的路由 ID，保持唯一
spring.cloud.gateway.routes[0].id=my-gateway
# 目标服务地址
spring.cloud.gateway.routes[0].uri=http://httpbin.org
# 路由条件
spring.cloud.gateway.routes[0].predicates[0]=Path=/get
# GatewayFilter（添加请求头）
spring.cloud.gateway.routes[0].filters[0]=AddRequestHeader=X-Request-Foo, Bar
# GatewayFilter（添加响应头）
spring.cloud.gateway.routes[0].filters[1]=AddResponseHeader=X-Response-Foo, Bar
```

也可以通过代码配置

```java
/**
 * 通过代码的形式配置路由
 * @param builder
 * @return
 */
@Bean
public RouteLocator myRoutes(RouteLocatorBuilder builder) {
	return builder.routes()
			.route(
					p -> p.path("/get")
					     .filters(f -> f.addRequestHeader("X-Request-Foo", "Bar").addResponseHeader("X-Response-Foo", "Bar"))
					     .uri("http://httpbin.org")
					)
			.build();
}
```

启动服务，访问 [http://localhost:8074/get](http://localhost:8074/get)

得到以下响应信息：

```json
{
  "args": {}, 
  "headers": {
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3", 
    "Accept-Encoding": "gzip, deflate, br", 
    "Accept-Language": "zh-CN,zh;q=0.9", 
    "Cache-Control": "max-age=0", 
    "Cookie": "Webstorm-2f8f75da=e0c5ee46-9276-490c-b32b-d5dc1483ca18; acw_tc=2760828015735472938194099e940a3c3ebc07316bcb1096abc6fefde61bf8; BD_UPN=12314353; BD_HOME=0", 
    "Forwarded": "proto=http;host=\"localhost:8074\";for=\"0:0:0:0:0:0:0:1:55395\"", 
    "Host": "httpbin.org", 
    "Sec-Fetch-Mode": "navigate", 
    "Sec-Fetch-Site": "none", 
    "Sec-Fetch-User": "?1", 
    "Upgrade-Insecure-Requests": "1", 
    "User-Agent": "Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/78.0.3904.97 Safari/537.36", 
    "X-Forwarded-Host": "localhost:8074", 
    "X-Request-Foo": "Bar"
  }, 
  "origin": "0:0:0:0:0:0:0:1, 183.14.135.71, ::1", 
  "url": "https://localhost:8074/get"
}
```

注意看添加了一个请求头："X-Request-Foo": "Bar"

响应头也添加了

![image](https://miansen.wang/assets/20191113171608.png)

通过这个例子可以知道，当 Gateway Handler Mapping 确定请求与哪个路由匹配之后，会将请求发送至 Gateway web handler 进行 GatewayFilter 拦截。 GatewayFilter 分为请求 filter 和 响应 filter，前者可以对请求信息过滤，后者可以对响应信息过滤。

#### 自定义 GatewayFilter

源码下载：[https://github.com/miansen/SpringCloud-Learn](https://github.com/miansen/SpringCloud-Learn)