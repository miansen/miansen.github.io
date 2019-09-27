---
layout: post
title: 四、SpringCloud-高可用的服务注册中心
date: 2019-09-27
categories: SpringCloud
tags: SpringCloud
author: 龙德
---

* content
{:toc}

前面三篇写了服务注册中心和服务之间的调用方式。到目前为止所有的服务都是注册到一台注册中心，万一这台注册中心挂了，那么所有的服务不是都得挂了。

为了解决这个问题，我们可以搭建一个注册中心集群。

把原先的 `spring-cloud-eureka-server` 工程复制一份，命名为 `spring-cloud-eureka-server-hign`，将启动类的名字修改为 `EurekaServerHignApplication`。

搭建集群应该有多个程序的，但是为了简单起见，可以利用 `spring.profiles.active` 属性，指定不同的配置文件达到启动多个程序的效果。

（1）新建 `application-dev1.properties` 配置文件

```properties
#指定服务名称
spring.application.name=spring-cloud-eureka-server-hign
#指定运行端口
server.port=8080
#指定主机地址
eureka.instance.hostname=localhost
#指定是否要从注册中心获取服务（注册中心不需要开启）
eureka.client.fetch-registry=false
#指定是否要注册到注册中心（注册中心不需要开启）
eureka.client.register-with-eureka=false
#关闭保护模式
eureka.server.enable-self-preservation=false
#配置注册中心地址
eureka.client.service-url.defaultZone=http://localhost:8081/eureka,http://localhost:8082/eureka
```

`application-dev1.properties` 指定了注册中心的运行端口是 `8080`，但是它同时也向 `8081` 和 `8082` 这两个端口的注册中心注册。

（2）新建 `application-dev2.properties` 配置文件

```properties
#指定服务名称
spring.application.name=spring-cloud-eureka-server-hign
#指定运行端口
server.port=8081
#指定主机地址
eureka.instance.hostname=localhost
#指定是否要从注册中心获取服务（注册中心不需要开启）
eureka.client.fetch-registry=false
#指定是否要注册到注册中心（注册中心不需要开启）
eureka.client.register-with-eureka=false
#关闭保护模式
eureka.server.enable-self-preservation=false
#配置注册中心地址
eureka.client.service-url.defaultZone=http://localhost:8080/eureka,http://localhost:8082/eureka
```

`application-dev2.properties` 指定了注册中心的运行端口是 `8081`，但是它同时也向 `8080` 和 `8082` 这两个端口的注册中心注册。


（3）新建 `application-dev3.properties` 配置文件

```properties
#指定服务名称
spring.application.name=spring-cloud-eureka-server-hign
#指定运行端口
server.port=8082
#指定主机地址
eureka.instance.hostname=localhost
#指定是否要从注册中心获取服务（注册中心不需要开启）
eureka.client.fetch-registry=false
#指定是否要注册到注册中心（注册中心不需要开启）
eureka.client.register-with-eureka=false
#关闭保护模式
eureka.server.enable-self-preservation=false
#配置注册中心地址
eureka.client.service-url.defaultZone=http://localhost:8080/eureka,http://localhost:8081/eureka
```

`application-dev3.properties` 指定了注册中心的运行端口是 `8082`，但是它同时也向 `8080` 和 `8081` 这两个端口的注册中心注册。

![image](https://miansen.wang/assets/20190927105348.png)

（4）指定启动参数 `--spring.profiles.active=dev1`，分别改成 `dev1`、`dev2`、`dev3` 启动三次。

![image](https://miansen.wang/assets/20190927174017.png)

访问 `http://localhost:8080`，如图：

![image](https://miansen.wang/assets/20190927144956.png)

源码下载：[https://github.com/miansen/SpringCloud-Learn](https://github.com/miansen/SpringCloud-Learn)