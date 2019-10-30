---
layout: post
title: 七、SpringCloud-服务配置中心 Spring-Cloud-Config
date: 2019-10-22
categories: SpringCloud
tags: SpringCloud Spring-Cloud-Config
author: 龙德
---

* content
{:toc}

> Spring Cloud Config 是用来为分布式系统中的基础设施和微服务应用提供集中化的外部配置支持，它分为服务端与客户端两个部分。其中服务端也称为分布式配置中心，它是一个独立的微服务应用，用来连接配置仓库并为客户端提供获取配置信息、加密 / 解密信息等访问接口；而客户端则是微服务架构中的各个微服务应用或基础设施，它们通过指定的配置中心来管理应用资源与业务相关的配置内容，并在启动的时候从配置中心获取和加载配置信息。

## 服务端

新建一个子工程，命名为 spring-cloud-config-server

引入依赖

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

application.properties

```properties
# 指定服务名称
spring.application.name=spring-cloud-config-server
# 指定运行端口
server.port=8072

# 配置 Git 仓库地址
spring.cloud.config.server.git.uri=https://github.com/miansen/SpringCloud-Learn
# 配置文件所在的具体目录（也可以用通配符 /**，因为 SpringCloudConfig 只根据配置文件名找配置）
spring.cloud.config.server.git.searchPaths=config
# 配置仓库的分支（默认是master）
spring.cloud.config.label=master
# 配置 Git 仓库的用户名（如果 Git 仓库为公开仓库，可以不填写用户名和密码，如果是私有仓库需要填写）
spring.cloud.config.server.git.username=
# 访问 Git 仓库的用户密码
spring.cloud.config.server.git.password=
```

在远程 Git 仓库上新建几个配置文件，如图所示：

![image](https://miansen.wang/assets/20191022182938.png)

可以看到我新建了4个配置文件

- application-dev.properties
- application-dev.yml
- application-prod.properties
- application-prod.yml

这4个配置文件的命名都有一定的规律，可以概括为：{应用名} - {环境名} . {格式名}

为什么要这样的命名格式呢，随便命名不行吗？

其实这样命名是跟 http 请求地址和资源文件的映射有关系，SpringCloudConfig 会根据 http 请求地址去查找文件。

查找规则如下：

- / {应用名} / {环境名} / {分支名}
- / {应用名} - {环境名} . {格式名}
- / {分支名} / {应用名} - {环境名} . {格式名}

接着启动工程

按照上面的查找规则，想要获取 application-dev.properties 配置文件，对应的访问地址如下：

- http://localhost:8072/application/dev/master
- http://localhost:8072/application-dev.properties
- http://localhost:8072/master/application-dev.properties

![image](https://miansen.wang/assets/20191022184555.png)

可以看到返回了配置文件的内容，证明配置服务端可以从远程 Git 仓库获取到配置信息。

## 客户端

客户端就是各个微服务应用了，为了演示简单，我这里新建一个子工程，命名为 spring-cloud-config-client

pom.xml 引入下面的依赖

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

注意客户端引入的是 `spring-cloud-starter-config` 包

新建 bootstrap.properties 配置文件，boostrap 由父 ApplicationContext 加载，比 applicaton 优先加载

```properties
# 指定服务名称
spring.application.name=spring-cloud-config-client
# 指定运行端口
server.port=8073

# 服务配置中心的地址
spring.cloud.config.uri=http://localhost:8072/
# 指定配置文件的分支
spring.cloud.config.label=master
# 指定配置文件的环境
spring.cloud.config.profile=dev
```

启动类把从服务端获取到的配置输出，以便能直观的看到

```java
@RestController
@SpringBootApplication
public class SpringCloudConfigClientApplication {

	public static void main(String[] args) {
		SpringApplication.run(SpringCloudConfigClientApplication.class, args);
	}
	
	@Value("${key}")
    private String key;

    @GetMapping("/key")
    public String getKey() {
        return key;
    }
}
```

接着启动客户端，访问 `http://localhost:8073/key`，输出内容如下：

![image](https://miansen.wang/assets/20191030192739.png)

以后客户端切换配置时只需要修改 spring.cloud.config.profile 的值即可