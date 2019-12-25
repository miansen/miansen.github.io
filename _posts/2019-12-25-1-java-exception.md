---
layout: post
title: Java 的异常
date: 2019-12-25
categories: Java
tags: Exception
author: 龙德
---

* content
{:toc}

## 异常的分类

Java 的异常分为两类，受检异常和非受检异常。也可以叫运行时异常和非运行时异常。都是一样的意思。

先看看受检异常

```java
public void foo() throws Exception {
		System.out.println("hello world!");
	}
```

throws 关键字作用于方法名后面，作用是抛出异常。

这里抛出了 Exception 异常，这是个受检异常。也就是说调用方必须对这个异常进行处理，否则编译不会通过。

![](https://i.loli.net/2019/12/25/akEOvBxd5IXi8St.png)

这是 Eclipse 给出的提示，要么继续抛出异常，交给上层调用者处理，要么就在方法里处理。

然后是非受检异常

```java
public void foo() throws RuntimeException {
		System.out.println("hello world!");
	}
```

RuntimeException 就是一个非受检异常，从名字就可以知道，这是一个在程序运行时才有可能会发生的异常。

由于是运行时才会发生的异常，所以非受检异常不强制要求做处理。