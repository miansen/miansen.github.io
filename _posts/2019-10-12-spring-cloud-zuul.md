---
layout: post
title: 六、SpringCloud-路由器和过滤器 Zuul
date: 2019-10-12
categories: Spring全家桶
tags: SpringCloud
author: 龙德
---

* content
{:toc}

路由是微服务架构中的一部分，例如访问 `/` 映射到首页，访问 `/user/**` 映射到用户服务，访问 `/order/**` 映射到订单服务。Zuul 是 Netflix 的基于 JVM 的路由器和服务器端负载均衡器，默认和 Ribbon 结合实现了负载均衡的功能。

## 路由

新建一个工程，命名为 spring-cloud-zuul-server

pom.xml 引入下面的依赖

```xml
<!-- 引入 web 依赖 -->
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
<!-- 引入 eureka server 依赖 -->
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
<!-- 引入 zuul 依赖 -->
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-zuul</artifactId>
</dependency>
```

启动类添加 @EnableZuulProxy 注解

```java
@EnableZuulProxy
@EnableEurekaClient
@SpringBootApplication
public class ZuulServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(ZuulServerApplication.class, args);
	}

}
```

application.properties

```properties
#指定服务名称
spring.application.name=spring-cloud-zuul-server
#指定运行端口
server.port=8070
#获取注册实例列表
eureka.client.fetch-registry=true
#注册到 Eureka 的注册中心
eureka.client.register-with-eureka=true
#配置注册中心地址
eureka.client.service-url.defaultZone=http://localhost:8080/eureka

# 配置 feign 路由
zuul.routes.feign.path=/feign/**
zuul.routes.feign.serviceId=spring-cloud-consumer-feign-hystrix

# 配置 ribbon 路由
zuul.routes.ribbon.path=/ribbon/**
zuul.routes.ribbon.serviceId=spring-cloud-consumer-ribbon-hystrix
```

这里配置了两个路由，第一个路由的名字是 feign，访问地址是 /feign/**，映射的服务是 spring-cloud-consumer-feign-hystrix，第二个同理。


接着分别启动 spring-cloud-eureka-server、spring-cloud-provider、spring-cloud-consumer-feign-hystrix、spring-cloud-consumer-ribbon-hystrix、spring-cloud-zuul-server 这5个工程。

如图所示：

![image](https://miansen.wang/assets/20191012152458.png)

假如没有 Zuul，看看之前我们是怎么访问的。

访问 spring-cloud-consumer-feign-hystrix 服务，则需要输入相应的地址 `http://localhost:8076/users/zhangsan`

访问 spring-cloud-consumer-ribbon-hystrix 服务，则需要输入相应的地址 `http://localhost:8077/users/zhangsan`

有多少个服务，就要维护多少个调用方式。

如图所示：

![image](https://miansen.wang/assets/zuul-1.png)

现在有了 Zuul，就不需要一个个的去维护了，统一起来全部交给 Zuul 管理。

访问 spring-cloud-consumer-feign-hystrix 服务，只需要输入我们配置的地址 `http://localhost:8070/feign/users/zhangsan`

访问 spring-cloud-consumer-ribbon-hystrix 服务，只需要输入我们配置的地址 `http://localhost:8070/ribbon/users/zhangsan`

如图所示：

![image](https://miansen.wang/assets/zuul-2.png)

## 负载均衡

服务往往是集群部署的，如果一个服务有多个实例，Zuul 也可以做到负载均衡。

分别再开启一个 spring-cloud-consumer-feign-hystrix  和 spring-cloud-consumer-ribbon-hystrix 工程

![image](https://miansen.wang/assets/20191012160951.png)

如图所示，spring-cloud-consumer-feign-hystrix 服务现在有两个实例，端口是 8075 和 8076

spring-cloud-consumer-ribbon-hystrix 服务也有两个实例，端口是 8077 和 8078

只需要在配置文件里加入以下参数就可以开启 Zuul 的负载均衡功能

```properties
# 配置 feign 路由
zuul.routes.feign.path=/feign/**
zuul.routes.feign.serviceId=spring-cloud-consumer-feign-hystrix
spring-cloud-consumer-feign-hystrix.ribbon.listOfServers=http://localhost:8075,http://localhost:8076

# 配置 ribbon 路由
zuul.routes.ribbon.path=/ribbon/**
zuul.routes.ribbon.serviceId=spring-cloud-consumer-ribbon-hystrix
spring-cloud-consumer-ribbon-hystrix.ribbon.listOfServers=http://localhost:8077,http://localhost:8078
```

## 过滤

Zuul 不仅是路由器，而且也是过滤器，可以做到安全校验的功能。

继续改造 spring-cloud-zuul-server 工程，新建一个类 MyFilter

```java
@Component
public class MyFilter extends ZuulFilter {

	/**
	 * 是否要过滤，可以写具体的逻辑进行判断。
	 * <p>我这里为 true，永远过滤。
	 */
	@Override
	public boolean shouldFilter() {
		return true;
	}

	/**
	 * 过滤的具体逻辑
	 */
	@Override
	public Object run() throws ZuulException {
		// 共享 RequestContext，上下文对象
		RequestContext ct = RequestContext.getCurrentContext();
		// 我这里只是简单的获取 token，然后判断是否为空
		String token = ct.getRequest().getParameter("token");
		if(token == null || "".equals(token)) {
			// 过滤该请求，不对其进行路由
			ct.setSendZuulResponse(false);
			// 返回错误代码
			ct.setResponseStatusCode(HttpStatus.UNAUTHORIZED.value());
			// 返回错误信息
			ct.setResponseBody("token must be not null");
		}
		return null;
	}

	/**
	 * 返回一个字符串代表过滤器的类型，在zuul中定义了四种不同生命周期的过滤器类型，具体如下
	 * <p>pre：路由之前
	 * <p>routing：路由之时
	 * <p>post： 路由之后
	 * <p>error：发送错误调用
	 */
	@Override
	public String filterType() {
		return FilterConstants.PRE_TYPE;
	}

	/**
	 * 过滤的顺序， 越小越靠前
	 */
	@Override
	public int filterOrder() {
		return FilterConstants.SERVLET_DETECTION_FILTER_ORDER - 1;
	}

}
```

访问 `http://localhost:8070/ribbon/users/zhangsan`，这时 token 为空，则会过滤掉该请求。

![image](https://miansen.wang/assets/20191012171140.png)

访问 `http://localhost:8070/ribbon/users/zhangsan?token=aaa`，则正常

![image](https://miansen.wang/assets/20191012171312.png)

源码下载：[https://github.com/miansen/SpringCloud-Learn](https://github.com/miansen/SpringCloud-Learn)