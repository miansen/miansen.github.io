---
layout: post
title: Java反射-Method的Invoke方法
date: 2019-09-05
categories: Java
tags: 反射 Method
author: 龙德
---

* content
{:toc}

**一个简简单单的 User 类**

```java
public class User {

	private String name;

	private int age;

	public User() {
	}

	public User(String name, int age) {
		this.name = name;
		this.age = age;
	}

	public void print() {
		System.out.println("name: " + this.name + ", age: " + this.age);
	}

	public void print(String sex, String addr) {
		System.out.println("name: " + this.name + ", age: " + this.age + ", sex: " + sex + ", addr: " + addr);
	}

	private void privateMethod() {
		System.out.println("private 方法");
	}

	protected void protectedMethod() {
		System.out.println("protected 方法");
	}

	void defaultMethod() {
		System.out.println("default 方法");
	}

	public static void staticMethod() {
		System.out.println("static 方法");
	}

	public final void finalMethod() {
		System.out.println("final 方法");
	}
}
```

**测试类**

```java
public class UserTest {

	public static void main(String[] args) throws Exception {
		// 1.先获取 User 类的 Class 对象
		Class userClass = User.class;
		
		// 2.通过 Class 对象获取 Method 对象，参数是方法名（String methodName）和参数类型（Class<?>... parameterTypes）
		
		// public 修饰的空参方法
		Method print1 = userClass.getMethod("print");
		
		// public 修饰的带参方法
		Method print2 = userClass.getMethod("print", String.class, String.class);
		
		// private 修饰的方法，抛出异常 java.lang.NoSuchMethodException
		// Method privateMethod = userClass.getMethod("privateMethod", null);
		
		// protected 修饰的方法，抛出异常 java.lang.NoSuchMethodException
		// Method protectedMethod = userClass.getMethod("protectedMethod", null);
		
		// default 修饰的方法，抛出异常 java.lang.NoSuchMethodException
		// Method defaultMethod = userClass.getMethod("defaultMethod", null);
		
		// static 方法
		Method staticMethod = userClass.getMethod("staticMethod", null);
		
		// final 方法
		Method finalMethod = userClass.getMethod("finalMethod", null);
		
		// 3.调用 Method 对象的 invoke 方法，将 User 类的实例对象和方法的参数传进去
		User user1 = new User("ZhangSan",10);
		User user2 = new User("LiSi",12);
		print1.invoke(user1);
		print2.invoke(user2,"男","中国");
		staticMethod.invoke(user2);
		finalMethod.invoke(user2);
		
		// 一般的方法调用
		System.out.println("========= 一般的方法调用 ========");
		user1.print();
		user2.print("男","中国");
		user2.staticMethod();
		user2.finalMethod();
	}
}
```

**控制台输出**

```
name: ZhangSan, age: 10
name: LiSi, age: 12, sex: 男, addr: 中国
static 方法
final 方法
========= 一般的方法调用 ========
name: ZhangSan, age: 10
name: LiSi, age: 12, sex: 男, addr: 中国
static 方法
final 方法
```

可以看到通过 Method 对象调用的 invoke() 方法跟对象.方法名的调用的结果是一样的。

**结论**

终于明白 JDK 的动态代理为什么能执行原对象的方法了，原来是将 Method 对象和方法参数 args 传进了 InvocationHandler 接口的 invoke(Object proxy, Method method, Object[] args) 方法里，然后通过调用 method.invoke(原对象实例，args) 方法就可以达到调用原对象方法的效果。

```java
public class ObjInterceptor implements InvocationHandler {

		@Override
		public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
			return method.invoke(obj, args);
		}
	}
```