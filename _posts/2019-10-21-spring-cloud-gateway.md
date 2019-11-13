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

先看看官网的简介

> Spring Cloud Gateway 是 Spring Cloud 的一个全新项目，该项目是基于 Spring 5.0，Spring Boot 2.0 和 Project Reactor 等技术开发的网关，它旨在为微服务架构提供一种简单有效的统一的 API 路由管理方式。
>
> Spring Cloud Gateway 作为 Spring Cloud 生态系统中的网关，目标是替代 Netflix Zuul，其不仅提供统一的路由方式，并且基于 Filter 链的方式提供了网关基本的功能，例如：安全，监控/指标，和限流。

## 概念

- Route（路由）：这是网关的基本构建块。它由一个 ID，一个目标 URI，一组断言和一组过滤器定义。如果断言为真，则路由匹配。
- Predicate（断言）：这是一个 Java 8 的 Predicate。我们可以使用它来匹配来自 HTTP 请求的任何内容，例如 Path、Host、Headers 等。
- Filter（过滤器）：GatewayFilter 的实例，我们可以使用它修改请求和响应。

## 快速上手

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
spring.cloud.gateway.routes[0].id=my-gateway
spring.cloud.gateway.routes[0].uri=http://httpbin.org
spring.cloud.gateway.routes[0].predicates[0]=Path=/get
```

配置文件定义了一个 router（注意是数组的形式），各个属性的含义如下：

- id：自定义的路由 ID，保持唯一
- uri：目标服务地址
- predicates：路由条件，Predicate 接受一个输入参数，返回一个布尔值结果

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

除了根据 Path 断言以外，还能根据时间、Cookie、Header、Host、Mthod 等断言，具体的可以看一下官方文档，写的很清楚。

[https://cloud.spring.io/spring-cloud-static/spring-cloud-gateway/2.0.0.RELEASE/single/spring-cloud-gateway.html](https://cloud.spring.io/spring-cloud-static/spring-cloud-gateway/2.0.0.RELEASE/single/spring-cloud-gateway.html)

## Filter

Spring Cloud Gateway 根据作用范围划分为 GatewayFilter 和 GlobalFilter，二者区别如下：

- GatewayFilter：需要通过 spring.cloud.routes.filters 配置在具体路由下，只作用在当前路由上
- GlobalFilter：全局过滤器，不需要在配置文件中配置，作用在所有的路由上

官方提供了很多 GatewayFilter 和 GlobalFilter，可以在这里查看： [https://cloud.spring.io/spring-cloud-static/spring-cloud-gateway/2.0.0.RELEASE/single/spring-cloud-gateway.html](https://cloud.spring.io/spring-cloud-static/spring-cloud-gateway/2.0.0.RELEASE/single/spring-cloud-gateway.html)

### GatewayFilter

先看看官方提供的 GatewayFilter

![image](/assets/GatewayFilter.jpg)

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

流程是这样的：当客户端请求到达 Spring Cloud Gateway 后，Gateway Handler Mapping 会将其拦截，根据 predicates 确定请求与哪个路由匹配，如果匹配成功，则会将请求发送至 Gateway web handler。Gateway web handler 处理请求会经过一系列 "pre" 类型的 Filter，这里是 AddRequestHeaderGatewayFilterFactory，添加了一个请求头 X-Request-Foo: Bar 。服务响应后再执行一系列的 "post" 类型的 Filter，这里是 AddResponseHeaderGatewayFilterFactory，添加了一个响应头 X-Response-Foo: Bar。 

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

![image](/assets/20191113171608.png)

通过这个例子可以知道，当 Gateway Handler Mapping 确定请求与哪个路由匹配之后，会将请求发送至 Gateway web handler 进行 GatewayFilter 拦截。 GatewayFilter 分为请求 filter 和 响应 filter，前者可以对请求信息过滤，后者可以对响应信息过滤。

#### 自定义 GatewayFilter

源码下载：[https://github.com/miansen/SpringCloud-Learn](https://github.com/miansen/SpringCloud-Learn)