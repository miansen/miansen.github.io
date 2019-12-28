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

## 异常分类

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

## 异常堆栈

```java
public void controller() {
	service();
}

public void service() {
	dao();
}

public void dao() {
	System.out.println(1 / 0);
}

public static void main(String[] args) {
	ExceptionTest exceptionTest = new ExceptionTest();
	exceptionTest.controller();
}
```

控制台输出的异常信息：

```java
Exception in thread "main" java.lang.ArithmeticException: / by zero
	at com.test.ExceptionTest.dao(ExceptionTest.java:31)
	at com.test.ExceptionTest.service(ExceptionTest.java:25)
	at com.test.ExceptionTest.controller(ExceptionTest.java:19)
	at com.test.ExceptionTest.main(ExceptionTest.java:37)
```

异常堆栈的输出跟方法调用一样，也是先进后出的结构。main 方法最先调用，异常最后输出。dao 方法最后调用，异常最先输出。

## 异常链

捕捉住异常后，将老异常当做新异常的构造函数参数创建新异常并继续抛出，这样就形成了异常链。

```java
public void controller() throws Exception {
	try {
		service();
	} catch (Exception e) {
		throw new Exception("controller 层抛出的异常", e);
	}
}

public void service() throws Exception {
	try {
		dao();
	} catch (Exception e) {
		throw new Exception("service 层抛出的异常", e);
	}
}

public void dao() throws Exception {
	try {
		System.out.println(1/0);
	} catch (Exception e) {
		throw new Exception("dao 层抛出的异常", e);
	}
}
	
public static void main(String[] args) {
	ExceptionTest exceptionTest = new ExceptionTest();
	try {
		exceptionTest.controller();
	} catch (Exception e) {
		e.printStackTrace();
	}
}
```

控制台输出的异常信息：

```java
java.lang.Exception: controller 层抛出的异常
	at com.test.ExceptionTest.controller(ExceptionTest.java:21)
	at com.test.ExceptionTest.main(ExceptionTest.java:44)
Caused by: java.lang.Exception: service 层抛出的异常
	at com.test.ExceptionTest.service(ExceptionTest.java:29)
	at com.test.ExceptionTest.controller(ExceptionTest.java:19)
	... 1 more
Caused by: java.lang.Exception: dao 层抛出的异常
	at com.test.ExceptionTest.dao(ExceptionTest.java:37)
	at com.test.ExceptionTest.service(ExceptionTest.java:27)
	... 2 more
Caused by: java.lang.ArithmeticException: / by zero
	at com.test.ExceptionTest.dao(ExceptionTest.java:35)
	... 3 more
```

异常链也是先进后出的结构，新异常最先输出，老异常最后输出。也就是说第一个异常信息就是异常抛出的地方，最后一个异常信息就是产生异常的源头。