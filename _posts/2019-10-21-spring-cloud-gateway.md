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
- Predicate（断言）：这是一个 Java 8 的 Predicate。输入类型是一个 ServerWebExchange。我们可以使用它来匹配来自 HTTP 请求的任何内容，例如 headers 或参数。
- Filter（过滤器）：这是org.springframework.cloud.gateway.filter.GatewayFilter的实例，我们可以使用它修改请求和响应。

## 快速上手

新建一个子工程，命名为 spring-cloud-gateway

引入依赖

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

application.propertise

```propertise
spring.application.name=spring-cloud-gateway
server.port=8074
spring.cloud.gateway.routes[0].id=my-gateway
spring.cloud.gateway.routes[0].uri=https://www.baidu.com
spring.cloud.gateway.routes[0].predicates[0]=Path=/users/zhangsan
```

配置文件定义了一个 router（注意是数组的形式），各个属性的含义如下：

- id：自定义的路由 ID，保持唯一
- uri：目标服务地址
- predicates：路由条件，Predicate 接受一个输入参数，返回一个布尔值结果

上面这段配置的意思是，配置了一个 id 为 my-gateway 的路由规则，当访问地址 [http://localhost:8074/users/zhangsan](http://localhost:8074/users/zhangsan) 时会自动转发到 [https://www.baidu.com](https://www.baidu.com)

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
						p -> p.path("/users/zhangsan").uri("https://www.baidu.com")
						)
				.build();
	}
```

application.propertise 配置路由和代码配置路由选择其中一个就好了，个人推荐 application.propertise 的形式配置。

启动服务，访问 [http://localhost:8074/users/zhangsan](http://localhost:8074/users/zhangsan) 应该会跳转到 [https://www.baidu.com](https://www.baidu.com)

源码下载：[https://github.com/miansen/SpringCloud-Learn](https://github.com/miansen/SpringCloud-Learn)