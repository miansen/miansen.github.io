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

## 监控与仪表盘

### Ribbon 工程的监控

在 spring-cloud-consumer-ribbon-hystrix 工程的启动类里注入一个 bean 来初始化监控的 Servlet

```java
/**
 * 低版本直接启动即可使用 http://ip:port/hystrix.stream 查看监控信息
 * 高版本需要添加本方法方可使用 http://ip:port/{urlMappings} 查看监控信息
 * @return
 */
@Bean
public ServletRegistrationBean getServlet() {
	HystrixMetricsStreamServlet streamServlet = new HystrixMetricsStreamServlet();
    ServletRegistrationBean registrationBean = new ServletRegistrationBean(streamServlet);
    registrationBean.setLoadOnStartup(1);
    registrationBean.addUrlMappings("/actuator/hystrix.stream");
    registrationBean.setName("HystrixMetricsStreamServlet");
    return registrationBean;
}
```

启动 spring-cloud-consumer-ribbon-hystrix 工程

访问 `http://localhost:8077/actuator/hystrix.stream`

![image](https://miansen.wang/assets/20191009165150.png)

可以看到浏览器一直处于请求的状态，并且页面一直在打印 ping。

因为此时项目中注解了 @HystrixCommand 的方法还没有执行，因此也没有任何的监控数据。

访问 `http://localhost:8077/users/zhangsan`后，再次访问 `http://localhost:8077/actuator/hystrix.stream`，就可以看到监控数据了。

![image](https://miansen.wang/assets/20191009165531.png)

不过这些数据都是以文件形式展示的，很难一眼看出系统当前的运行状态。

### Feign 工程的监控

引入一个新的依赖

```xml
<!-- 因为需要监控功能，所以引入 Hystrix 依赖-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

启动类添加注解 `@EnableCircuitBreaker`，并且同样注入一个 bean 来初始化监控的 Servlet

启动后访问 `http://localhost:8076/actuator/hystrix.stream` 同样可以监控到数据。

### 仪表盘

上面的监控得到的数据都是以文件形式展示的，很难一眼看出系统当前的运行状态。

Hystrix 的主要优点之一是它收集关于每个 HystrixCommand 的一套指标，并通过 Hystrix 仪表盘有效的显示每个断路器的运行状况。

所以我们可以通过仪表盘来可视化的监控每个断路器的运行状况。

![image](https://miansen.wang/assets/Hystrix.png)

仪表盘长这个样子

新建一个子工程，命名为 spring-cloud-hystrix-dashboard

引入下面的依赖：

```xml
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
<!-- 引入仪表盘依赖 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
</dependency>
```

application.properties

```properties
#指定服务名称
spring.application.name=spring-cloud-hystrix-dashboard
#指定运行端口
server.port=8072
#获取注册实例列表
eureka.client.fetch-registry=true
#注册到 Eureka 的注册中心
eureka.client.register-with-eureka=true
#配置注册中心地址
eureka.client.service-url.defaultZone=http://localhost:8080/eureka
```

然后启动类添加注解 `@EnableHystrixDashboard`，开启仪表盘

访问 `http://localhost:8072/hystrix`

可以看到这个界面

![image](https://miansen.wang/assets/20191009163044.png)

第一个输入框，是要监控的地址，第二个是轮训时间，第三个是仪表盘页面的标题

我们在这3个框里分别输入 `http://localhost:8077/actuator/hystrix.stream`，2000，RibbonDashboard，然后点击 Monitor Stream，就可以直观的监控到服务运行的状态了。

![image](https://miansen.wang/assets/20191009170427.png)

想要监控其他的服务只需要把监控地址换了就行。

### Turbine 集群监控

上面的监控只是监控一个单体应用，但在实际应用中，程序往往是集群部署的，所以我们需要对集群进行监控，这时候可以采用 Turbine 进行集群监控。

改造 spring-cloud-hystrix-dashboard 工程

引入新的依赖

```xml
<!-- 引入 Turbine 依赖 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-turbine</artifactId>
</dependency>
```

application.properties 新增几个参数

```properties
# 表示要聚合的服务，多个用逗号隔开
turbine.app-config=spring-cloud-consumer-ribbon-hystrix,spring-cloud-consumer-feign-hystrix
# 表示集群的名字为default 
turbine.clusterNameExpression= "default"
# 同一主机上的服务通过host和port的组合来进行区分
turbine.combine-host-port=true
```

启动类新增注解 `@EnableTurbine`，开启集群监控面板。

启动注册中心 spring-cloud-eureka-server

启动服务 spring-cloud-consumer-ribbon-hystrix，且这个服务启动两个实例

启动集群监控 spring-cloud-hystrix-dashboard

访问集群监控的地址 `http://localhost:8072/hystrix`

![image](https://miansen.wang/assets/20191009182315.png)

分别输入 `http://localhost:8072/turbine.stream`，2000，Turbine，点击 Monitor Stream

就可以看到集群监控的页面了

![image](https://miansen.wang/assets/20191009182456.png)

源码下载：[https://github.com/miansen/SpringCloud-Learn](https://github.com/miansen/SpringCloud-Learn)