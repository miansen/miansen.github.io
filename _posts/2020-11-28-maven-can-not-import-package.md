---
layout: post
title: 解决 Maven 引入依赖却无法导包的问题
date: 2020-11-28
categories: 杂七杂八
tags: Maven
author: 龙德
---

* content
{:toc}

用 Maven 引入 jar 包，无法导包，编译报错。

![image](https://miansen.wang/assets/maven-scope-0.png)

![image](https://miansen.wang/assets/maven-scope-1.png)

![image](https://miansen.wang/assets/maven-scope-2.png)

![image](https://miansen.wang/assets/maven-scope-3.png)

可以看到 jar 包确实引入了，版本号没问题，类名和包名也没问题，究竟是什么原因无法导包呢？

遇到这种问题，可能是 Maven 的 scope 没有设置正确。对于 h2 数据库驱动这个 jar 包我定义的 scope 是 runtime，而 runtime 引入的 jar 包，应用代码里不能直接使用，用了无法通过编译，只能通过全限定名反射之类的方式来用。

我把 scope 改成 compile（不写的话默认也是这个），再刷新 Maven 就可以通过编译了。所以遇到这种问题，除了要检查 jar 包是否引入，包名是否正确以外，还要检查 scope 是否设置正确。

下面是对 scope 的简单总结：

- compile（编译）：可以直接 import 进来用，编译没问题。打包后出现在你的项目的 war 包里。
- provided（已提供）：可以直接 import 进来用，编译没问题。打包后不会出现在你的项目的 war 包里。这种一般是由容器（比如 Tomcat）提供。
- runtime（运行时）：不可以直接 import 进来用，否则编译报错，只能通过反射之类的方式来用。打包会出现在你的项目的 war 包里。
- test（测试）：不可以直接 import 进来用，否则编译报错。只能在测试编译和测试运行阶段可用。打包后不会出现在你的项目的 war 包里。
- system（系统）：system 范围依赖与 provided 类似，但是你必须显式的提供一个对于本地系统中 JAR 文件的路径。这么做是为了允许基于本地对象编译，而这些对象是系统类库的一部分。这样的构建应该是一直可用的，Maven  也不会在仓库中去寻找它。如果你将一个依赖范围设置成系统范围，你必须同时提供一个 systemPath 元素。注意该范围是不推荐使用的（建议尽量去从公共或定制的 Maven 仓库中引用依赖）。示例如下：

```xml
<project>
  <dependencies>
    <dependency>
      <groupId>sun.jdk</groupId>
      <artifactId>tools</artifactId>
      <version>1.5.0</version>
      <scope>system</scope>
      <systemPath>${java.home}/../lib/tools.jar</systemPath>
    </dependency>
    ...
  </dependencies>
</project>
```