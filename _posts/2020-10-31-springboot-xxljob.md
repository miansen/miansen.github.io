---
layout: post
title: springboot 集成 xxl-job 报错
date: 2020-10-31
categories: SpringBoot
tags: SpringBoot xxl-job
author: 龙德
---

* content
{:toc}

我的 springboot 版本是 2.0.5，xxl-job 版本是 2.0.1。

报错信息如下：

```java
***************************
APPLICATION FAILED TO START
***************************

Description:

An attempt was made to call the method com.xxl.job.core.executor.impl.XxlJobSpringExecutor.setAppname(Ljava/lang/String;)V but it does not exist. Its class, com.xxl.job.core.executor.impl.XxlJobSpringExecutor, is available from the following locations:

    jar:file:/D:/maven_repositor/com/xuxueli/xxl-job-core/2.0.1/xxl-job-core-2.0.1.jar!/com/xxl/job/core/executor/impl/XxlJobSpringExecutor.class

It was loaded from the following location:

    file:/D:/maven_repositor/com/xuxueli/xxl-job-core/2.0.1/xxl-job-core-2.0.1.jar


Action:

Correct the classpath of your application so that it contains a single, compatible version of com.xxl.job.core.executor.impl.XxlJobSpringExecutor
```

看这个错误信息应该是找不到 XxlJobSpringExecutor 类的 setAppname() 方法。

我打开 XxlJobSpringExecutor 类，发现只有 setAppName() 方法，并没有 setAppname() 方法。而且我的代码里也没有调 setAppname() 方法，而是调 setAppName() 方法。那么这个 setAppname() 方法是在哪里调用的呢？

既然我的代码里没有调用，很有可能是第三方 jar 包调用了，而且这个 jar 很可能是 spring boot starter，只有这样它才能在 springboot 启动时初始化，在初始化方法里调用 setAppname() 方法，从而抛出上述错误。

根据这个思路，我检查了 pom.xml 文件，发现这个带有 `job` 字样的依赖很可疑。

![image](https://miansen.wang/assets/springboot-xxljob-1.png)

点进去一看，发现它果然依赖了 xxl-job，而且版本比较新，是 2.2.0 版本。

![image](https://miansen.wang/assets/springboot-xxljob-2.png)

我找到这个 jar 包，打开 spring.factories 文件，找到初始化的方法。

![image](https://miansen.wang/assets/springboot-xxljob-3.png)

![image](https://miansen.wang/assets/springboot-xxljob-4.png)

可以看到这个 jar 包的初始化类是 XxlJobAutoConfiguration。我们打开这个 XxlJobAutoConfiguration 类，发现这个类果然调用了 setAppname() 方法。

![image](https://miansen.wang/assets/springboot-xxljob-5.png)

所以结论是：我在自己的 pom.xml 文件里引入版本为 2.0.1 的 xxl-job，这个版本只有 setAppName() 方法，并没有 setAppname() 方法。而第三方 jar 包引入的是版本为 2.2.0 的 xxl-job，这个最新的版本是有 setAppname() 方法的。但是我的版本把第三方 jar 包的覆盖了，所以第三方 jar 包的代码里调用 setAppname() 方法会报错，提示找不到这个方法！

解决方法：去掉我自己的 xxl-job 依赖，用第三方 jar 包的就好了。