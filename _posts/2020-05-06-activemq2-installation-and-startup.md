---
layout: post
title: 2.学习ActiveMQ-安装与启动
date: 2020-05-06
categories: ActiveMQ
tags: ActiveMQ
author: 龙德
---

* content
{:toc}

## 安装

从官网 [http://activemq.apache.org](http://activemq.apache.org) 可以下载 ActiveMQ，现在最新版本是 5.x，我们下载 Windows 平台的。

![image](https://miansen.wang/assets/20200506145007.png)

![image](https://miansen.wang/assets/20200506145043.png)

![image](https://miansen.wang/assets/20200506150402.png)

ActiveMQ 各个目录的作用：

> bin：ActiveMQ的启动脚本activemq.bat，注意分32、64位。
> 
> conf：配置文件，重点关注的是activemq.xml、jetty.xml、jetty-realm.properties。在登录ActiveMQ Web控制台需要用户名、密码信息；在JMS CLIENT和ActiveMQ进行何种协议的连接、端口是什么等这些信息都在上面的配置文件中可以体现。
> 
> data：ActiveMQ进行消息持久化存放的地方，默认采用的是kahadb，当然我们可以采用leveldb，或者采用JDBC存储到MySQL，或者干脆不使用持久化机制。
> 
> webapps：ActiveMQ自带Jetty提供Web管控台。
> 
> lib：ActiveMQ为我们提供了分功能的JAR包，当然也提供了activemq-all-5.14.4.jar。

## 启动

进入 bin/win64 目录，直接运行 activemq.bat，就可以启动 ActiveMQ。

![image](https://miansen.wang/assets/20200506151508.png)

访问首页：[http://localhost:8161/index.html](http://localhost:8161/index.html)

![image](https://miansen.wang/assets/20200506151657.png)

看到这个页面就说明已经成功启动 ActiveMQ 了。

访问控制台：[http://localhost:8161/admin/index.jsp](http://localhost:8161/admin/index.jsp)

访问控制台需要输入账号和密码，账号和密码配置在 conf 目录下的 jetty-realm.properties 文件里。

![image](https://miansen.wang/assets/20200506152709.png)

端口号配置在 jetty.xml 文件里。

![image](https://miansen.wang/assets/20200506152848.png)

输入账号密码就可以访问控制台了。

![image](https://miansen.wang/assets/20200506152957.png)

至此，ActiveMQ 的安装和启动就完成了。