---
layout: post
title: 四、SpringCloud-高可用的服务注册中心
date: 2019-09-27
categories: Spring全家桶
tags: SpringCloud
author: 龙德
---

* content
{:toc}

前面三篇写了服务注册中心和服务之间的调用方式。到目前为止所有的服务都是注册到一台注册中心，万一这台注册中心挂了，那么所有的服务不是都得挂了。

为了解决这个问题，我们可以搭建一个注册中心集群。

## 准备环境

集群意味着要有多台服务器，虽然可以通过改 `host` 文件的形式来搭建集群。不过我并不推荐这种形式，因为实际中不可能去改 `host` 文件的。所以我准备了3台虚拟机，地址分别是 192.168.8.4、192.168.8.6、192.168.8.8

## 搭建集群

首先把 `spring-cloud-eureka-server` 工程复制一份，命名为 `spring-cloud-eureka-server-hign`，将启动类的名字修改为 `EurekaServerHignApplication`。


（1）新建 `application-dev1.properties` 配置文件

```properties
#指定服务名称
spring.application.name=spring-cloud-eureka-server-hign
#指定运行端口
server.port=8080
#指定主机地址
eureka.instance.hostname=192.168.8.4
#应用程序使用IP地址的方式向Eureka注册
eureka.instance.prefer-ip-address=true
#指定是否要从注册中心获取服务（注册中心不需要开启）
eureka.client.fetch-registry=false
#指定是否要注册到注册中心（注册中心不需要开启）
eureka.client.register-with-eureka=false
#关闭保护模式
eureka.server.enable-self-preservation=false
#配置注册中心地址
eureka.client.service-url.defaultZone=http://192.168.8.6:8081/eureka,http://192.168.8.8:8082/eureka
```

`application-dev1.properties` 指定了注册中心的IP和端口 `192.168.8.4:8080`。它同时也向 `192.168.8.6:8081` 和 `192.168.8.8:8082` 这两个注册中心注册。

同理可得

`application-dev2.properties` 配置文件

```properties
#指定服务名称
spring.application.name=spring-cloud-eureka-server-hign
#指定运行端口
server.port=8081
#指定主机地址
eureka.instance.hostname=192.168.8.6
#应用程序使用IP地址的方式向Eureka注册
eureka.instance.prefer-ip-address=true
#指定是否要从注册中心获取服务（注册中心不需要开启）
eureka.client.fetch-registry=false
#指定是否要注册到注册中心（注册中心不需要开启）
eureka.client.register-with-eureka=false
#关闭保护模式
eureka.server.enable-self-preservation=false
#配置注册中心地址
eureka.client.service-url.defaultZone=http://192.168.8.4:8080/eureka,http://192.168.8.8:8082/eureka
```

`application-dev3.properties` 配置文件

```properties
#指定服务名称
spring.application.name=spring-cloud-eureka-server-hign
#指定运行端口
server.port=8082
#指定主机地址
eureka.instance.hostname=192.168.8.8
#应用程序使用IP地址的方式向Eureka注册
eureka.instance.prefer-ip-address=true
#指定是否要从注册中心获取服务（注册中心不需要开启）
eureka.client.fetch-registry=false
#指定是否要注册到注册中心（注册中心不需要开启）
eureka.client.register-with-eureka=false
#关闭保护模式
eureka.server.enable-self-preservation=false
#配置注册中心地址
eureka.client.service-url.defaultZone=http://192.168.8.4:8080/eureka,http://192.168.8.6:8081/eureka
```

![image](https://miansen.wang/assets/20190927105348.png)


（2）将 `spring-cloud-eureka-server-hign` 工程打成 jar 包，每台虚拟机上都部署一份

（3）启动程序

第一台虚拟机 192.168.8.4

```
java -jar spring-cloud-eureka-server-hign-0.0.1-SNAPSHOT.jar --spring.profiles.active=dev1
```

第二台虚拟机 192.168.8.6

```
java -jar spring-cloud-eureka-server-hign-0.0.1-SNAPSHOT.jar --spring.profiles.active=dev2
```

第三台虚拟机 192.168.8.8

```
java -jar spring-cloud-eureka-server-hign-0.0.1-SNAPSHOT.jar --spring.profiles.active=dev3
```


（6）分别访问，如图：

![image](https://miansen.wang/assets/20190928180750.jpg)

![image](https://miansen.wang/assets/20190928180913.jpg)

![image](https://miansen.wang/assets/20190928181016.jpg)

## 使用集群

还记得之前我们是如何注册的吗？没有集群的时候是这样写的

```properties
#指定服务名称
spring.application.name=spring-cloud-provider
#指定运行端口
server.port=8078
#获取注册实例列表
eureka.client.fetch-registry=true
#注册到 Eureka 的注册中心
eureka.client.register-with-eureka=true
#配置注册中心地址
eureka.client.service-url.defaultZone=http://localhost:8080/eureka
```

现在有了集群，就不能这么写，否则集群就没有意义了。

可以这么写：

```properties
#指定服务名称
spring.application.name=spring-cloud-provider
#指定运行端口
server.port=8078
#指定主机地址
eureka.instance.hostname=192.168.8.4
#应用程序使用IP地址的方式向Eureka注册
eureka.instance.prefer-ip-address=true
#获取注册实例列表
eureka.client.fetch-registry=true
#注册到 Eureka 的注册中心
eureka.client.register-with-eureka=true
#配置注册中心地址
eureka.client.service-url.defaultZone=http://192.168.8.4:8080/eureka,http://192.168.8.6:8081/eureka,http://192.168.8.8:8082/eureka
```

同理我们将 `spring-cloud-provider` 工程复制一份，命名为 `spring-cloud-provider-hign`，新建 `application-dev1.properties`、`application-dev2.properties`、`application-dev3.properties` 三个配置文件

将 `spring-cloud-provider` 工程打成 jar 包，每台虚拟机上都部署一份，然后分别启动工程。

访问如图：

![image](https://miansen.wang/assets/20190928191424.jpg)


源码下载：[https://github.com/miansen/SpringCloud-Learn](https://github.com/miansen/SpringCloud-Learn)