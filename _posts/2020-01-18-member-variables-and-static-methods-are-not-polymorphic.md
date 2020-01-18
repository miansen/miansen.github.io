---
layout: post
title: 成员变量和静态方法不具有多态性
date: 2020-01-18
categories: Java
tags: 多态
author: 龙德
---

* content
{:toc}

成员变量不具有多态性，请看下面的例子：

```java
public class Super {

	public int field = 0;

	public int getField() {
		return field;
	}

}
```

```java
public class Sub extends Super {

	public int field = 1;

	public int getField() {
		return field;
	}
	
	public int getSuperField() {
		return super.field;
	}
	
}
```

```java
public static void main(String[] args) {
		Super sup = new Sub();
		System.out.println("sup.field = " + sup.field + ", sup.getField() = " + sup.getField());
		Sub sub = new Sub();
		System.out.println("sub.field = " + sub.field + ", sub.getField() = " + sub.getField()
				+ ", sub.getSuperField() = " + sub.getSuperField());
	}
```

输出结果：

```
sup.field = 0, sup.getField() = 1
sub.field = 1, sub.getField() = 1, sub.getSuperField() = 0
```

本例中，Super 和 Sub 各有一个成员变量 field，当 Sub 继承 Super 后，Sub 实际包含两个名为 field 的成员变量：一个是它自己的，还有一个是从 Super 那里继承得来的。尽管这两个成员变量的名字相同，但是它们不是同一个变量，因为它们在内存中分配了不同的存储空间。当 Sub 对象向上转型为 Super 引用时，任何成员变量的访问操作都将由编译器解析。所以使用 Super 类型的引用访问 field 变量时，指向的是 Super 类中的 field。同理，使用 Sub 类型的引用访问 field 变量时，指向的是 Sub 类中的 field。你如果想访问从 Super 那里继承来的 field 时，必须使用 super 关键字，显示地调用 suepr.field。

虽然这看起来有些混淆，但是实际中，这些问题是不会发生的。首先，你通常会将所有的成员变量的访问权限都设置成 private，因此你无法直接访问它们，而是通过提供的 get 方法访问。另外，你通常不会对子类和父类的变量设置相同的名字，因为这种做法本身就容易令人混淆。

静态方法也不具有多态性，请看下面的例子：

```java
public class Super {

	public static String staticGet() {
		return "Super staticGet()";
	}
	
	public String dynamicGet() {
		return "Super dynamicGet()";
	}

}
```

```java
public class Sub extends Super {
	
	public static String staticGet() {
		return "Sub staticGet()";
	}
	
	public String dynamicGet() {
		return "Sub dynamicGet()";
	}
	
}
```

```java
public static void main(String[] args) {
		Super sup = new Sub();
		System.out.println(sup.staticGet());
		System.out.println(sup.dynamicGet());
	}
```

输出结果：

```
Super staticGet()
Sub dynamicGet()
```

静态方法是与类，而非某个对象相关联的，所以静态方法不具有多态性。