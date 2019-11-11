---
layout: post
title: 三、Spring Cloud-负载均衡 Ribbon
date: 2019-09-25
categories: SpringCloud
tags: Ribbon
author: 龙德
---

* content
{:toc}

## Ribbon 简介

目前主流的负载方案分为以下两种：

- 服务端负载均衡（Nginx）

服务实例的清单在服务端，服务器进行负载均衡算法分配

- 客户端负载均衡（Ribbon）

服务实例的清单在客户端，客户端进行负载均衡算法分配，Ribbon 就属于客户端自己做负载。

## 单独使用 Ribbon

我们使用 Ribbon 来实现一个最简单的负载均衡调用功能，接口就用 spring-cloud-provider 工程提供的 /users/{name} 接口，不过需要改动一下 controller 层

```java
@RestController
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

我们把服务名和端口也返回给客户端，目的是能更直观的看到负载均衡的效果。

因为要测试负载均衡，所有需要启动多个服务。这里我启动两个 spring-cloud-provider 服务，端口分别是 8071 和 8072。

接着新建一个 Maven 项目，命名为 ribbon-demo

pom.xml 引入以下依赖

```xml
<dependency>
	<groupId>com.netflix.ribbon</groupId>
	<artifactId>ribbon</artifactId>
	<version>2.2.2</version>
</dependency>
<dependency>
	<groupId>com.netflix.ribbon</groupId>
	<artifactId>ribbon-core</artifactId>
	<version>2.2.2</version>
</dependency>
<dependency>
	<groupId>com.netflix.ribbon</groupId>
	<artifactId>ribbon-loadbalancer</artifactId>
	<version>2.2.2</version>
</dependency>
```

随便新建一个测试类，在 main 方法里调用接口

```java
public static void main(String[] args) {
		// 服务列表
		List<Server> serverList = Lists.newArrayList(new Server("localhost", 8071), new Server("localhost", 8072));

		// 构建负载均衡器的实例
		BaseLoadBalancer loadBalancer = LoadBalancerBuilder.newBuilder().buildFixedServerListLoadBalancer(serverList);
		for (int i = 0; i < 5; i++) {
			// 构建执行负载均衡器命令的实例，用于中转请求到负载均衡器
			LoadBalancerCommand<String> loadBalancerCommand = LoadBalancerCommand.<String>builder()
					.withLoadBalancer(loadBalancer).build();
			Observable<String> observable = loadBalancerCommand.submit(new ServerOperation<String>() {

				@Override
				public Observable<String> call(Server server) {
					try {
						String addr = String.format("http://%s/users/zhangsan", server.getHostPort());
						System.out.println("调用地址：" + addr);
						URL url = new URL(addr);
						HttpURLConnection conn = (HttpURLConnection) url.openConnection();
						conn.setRequestMethod("GET");
						conn.connect();
						InputStream in = conn.getInputStream();
						byte[] data = new byte[in.available()];
						in.read(data);
						return Observable.just(new String(data));
					} catch (Exception e) {
						return Observable.error(e);
					}
				}
			});
			String result = observable.toBlocking().first();
			System.out.println("调用结果：" + result);
		}
	}
```

控制台输出如下

![image](/assets/20191108100003.png)

从输出的结果中可以看到，负载均衡起作用了，8071 调用了 2 次，8072 调用了 3 次。

## 在 Spring Cloud 中使用 Ribbon

Spring Cloud Ribbon 是一个基于 HTTP 和 TCP 的客户端负载均衡工具，它基于 Netflix Ribbon 实现。通过 Spring Cloud 的封装，可以让我们轻松地将面向服务的 REST 模版请求自动转换成客户端负载均衡的服务调用。
Spring Cloud Ribbon 虽然只是一个工具类框架，它不像服务注册中心、配置中心、API 网关那样需要独立部署，但是它几乎存在于每一个 Spring Cloud 构建的微服务和基础设施中。因为微服务间的调用，API 网关的请求转发等内容，实际上都是通过 Ribbon 来实现的。

在 Spring Cloud 中使用 Ribbon 也挺简单，因为 Spring Cloud 已经帮我们封装好了。甚至都不需要引入依赖，因为 Eureka 中已经引用了 Ribbon。

还记得我们之前在 spring-cloud-consumer 工程里注入了 RestTemplate 吗？

```java
@Configuration
public class BeanConfiguration {

	@Bean
	@LoadBalanced
	public RestTemplate getRestTemplate() {
		return new RestTemplate();
	}
}
```

其实这就已经实现了负载均衡的效果了，RestTemplate 加了一个 @LoadBalanced 注解之后，就可以结合 Eureka 来动态发现服务并进行负载均衡的调用。

我们再启动服务注册中心 spring-cloud-eureka-server 和服务消费者 spring-cloud-consumer 验证一下。

启动成功后，访问服务消费者 [http://localhost:8073/users/zhangsan](http://localhost:8073/users/zhangsan)，每次访问都会轮流的调用不同端口的消费提供者（注意看端口的变化）。

![image](/assets/ribbon3.gif)

### 自定义负载均衡策略

上面的例子采用的是默认的负载均衡策略，也就是轮训的方式。我们也可以自定义负载均衡的策略。

自定义负载均衡策略有两种方式

- 第一种方式

通过实现 IRule 接口自定义负载均衡策略，主要的选择服务逻辑在 choose 方法中。因为我们这里只是演示怎么自定义负载策略，所以没写选择的逻辑，而是直接返回服务列表中第一个服务。具体代码如下所示：

```java
public class MyRule implements IRule {

	private ILoadBalancer lb;

	@Override
	public Server choose(Object key) {
		List<Server> servers = lb.getAllServers();
		return servers.get(0);
	}

	@Override
	public void setLoadBalancer(ILoadBalancer lb) {
		this.lb = lb;
	}

	@Override
	public ILoadBalancer getLoadBalancer() {
		return lb;
	}

}
```

在 Spring Cloud 中，可通过配置的方式使用自定义的负载均衡策略。在 application.propertise 里加入以下配置：

```propertise
# 自定义的负载策略
spring-cloud-provider.ribbon.NFLoadBalancerRuleClassName=org.spring.cloud.consumer.config.MyRule
```

**注意 spring-cloud-provider 是调用的服务名称。**

也可以通过注解的方式使用自定义的负载均衡策略。

启动类上新增 @RibbonClient 注解

```java
@RibbonClient(name = "spring-cloud-provider", configuration = MyRule.class)
```

**这里的 spring-cloud-provider 同样也是调用的服务名称。**

- 第二种方式

通过继承 AbstractLoadBalancerRule 抽象类实现负载均衡的策略，主要的选择服务逻辑也是在 choose 方法中。具体的代码如下所示：

```java
public class MyRule extends AbstractLoadBalancerRule {

	@Override
	public Server choose(Object key) {
		List<Server> servers = getLoadBalancer().getAllServers();
		return servers.get(0);
	}

	@Override
	public void initWithNiwsConfig(IClientConfig clientConfig) {
		// TODO Auto-generated method stub
		
	}
}
```

同样的，可以通过配置的方式或者是注解的方式使用自定义的负载均衡策略。

重启后再次访问服务消费者 [http://localhost:8073/users/zhangsan](http://localhost:8073/users/zhangsan)，这时候端口应该不会发生变化，因为我们每次访问的都是服务列表中的第一个服务。

源码下载：[https://github.com/miansen/SpringCloud-Learn](https://github.com/miansen/SpringCloud-Learn)