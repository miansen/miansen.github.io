---
layout: post
title: Java ==、equals() 和 hashCode()
date: 2019-09-11
categories: Java
tags: equals hashCode
author: 龙德
---

* content
{:toc}

## ==

对于基本类型，== 比较的是字面量。

对于引用类型，== 比较的是地址值。

```java
String a = new String("abc");
String b = new String("abc");
String aa = "abc";
String bb = "abc";
// false，因为用 new 关键字创建 String 对象，每次都是新对象，地址值都不同
System.out.println(a == b);
// true，因为用字符串字面量创建 String 对象，都指向位于 Srping pool 中的同一个对象，地址值相同
System.out.println(aa == bb);
// true，基本类型比较的是字面量
System.out.println(42 == 42.0);
```

## equals()

分两种情况

1.没有重写 equals()，则比较的是地址值，相当于 == 比较。

```java
public class Test03 {

	public static void main(String[] args) {
		User user1 = new User("张三",15);
		User user2 = new User("张三",15);
		// false，因为 User 类没有重写 equals()，实际还是通过 == 比较
		System.out.printf("user1.equals(user2): %s \n",user1.equals(user2));
	}
	
	public static class User{
		String name;
		int age;
		public User(String name, int age) {
			this.name = name;
			this.age = age;
		}
	}
}
```

看一下 Object 中的 equals()

```java
public boolean equals(Object obj) {
    return (this == obj);
}
```

可以看到用的是 == 比较

2.重写了 equals()

我们可以在 equals() 里面先比较两个对象是否相等，如果不相等，再去比较内容是否相等。

```java
public class Test03 {

	public static void main(String[] args) {
		User user1 = new User("张三",15);
		User user2 = new User("张三",15);
		// true，因为重写了 equals()
		System.out.printf("user1.equals(user2): %s \n",user1.equals(user2));
	}
	
	public static class User{
		String name;
		int age;
		public User(String name, int age) {
			this.name = name;
			this.age = age;
		}

		@Override
		public boolean equals(Object obj) {
			if (this == obj)
				return true;
			if (obj == null)
				return false;
			if (getClass() != obj.getClass())
				return false;
			User other = (User) obj;
			if (age != other.age)
				return false;
			if (name == null) {
				if (other.name != null)
					return false;
			} else if (!name.equals(other.name))
				return false;
			return true;
		}
	}
}

```

String 类也重写了 equals()，所以我们可以直接拿来比较。

```java
public boolean equals(Object anObject) {
    if (this == anObject) {
        return true;
    }
    if (anObject instanceof String) {
        String anotherString = (String) anObject;
        int n = value.length;
        if (n == anotherString.value.length) {
            char v1[] = value;
            char v2[] = anotherString.value;
            int i = 0;
            while (n-- != 0) {
                if (v1[i] != v2[i])
                        return false;
                i++;
            }
            return true;
        }
    }
    return false;
}
```

## hashCode()

hashCode() 的作用是获取哈希码，也称为散列码；它实际上是返回一个 int 整数。这个哈希码的作用是确定该对象在哈希表中的索引位置。

通俗的将，在 Java 中，你要在本质是散列表的集合中用到该对象，如：HashMap，Hashtable，HashSet，那么 hashCode() 才起作用，其他情况下没用。

1.一般情况，只重写 equals()，没有重写 hashCode()

```java
public class Test03 {

	public static void main(String[] args) {
		User user1 = new User("张三",15);
		User user2 = new User("张三",15);
		System.out.printf("user1.equals(user2): %s, user1 hashCode: %d, user2 hashCode: %d \n",user1.equals(user2),user1.hashCode(),user2.hashCode());
		
	}
	
	public static class User{
		
		String name;
		
		int age;
		
		public User(String name, int age) {
			this.name = name;
			this.age = age;
		}

		@Override
		public boolean equals(Object obj) {
			if (this == obj)
				return true;
			if (obj == null)
				return false;
			if (getClass() != obj.getClass())
				return false;
			User other = (User) obj;
			if (age != other.age)
				return false;
			if (name == null) {
				if (other.name != null)
					return false;
			} else if (!name.equals(other.name))
				return false;
			return true;
		}
		
	}
}

```

输出：

```
user1.equals(user2): true, user1 hashCode: 382503496, user2 hashCode: 721139189 
```

可以看到 hashCode 不相等，并不影响 equals() 比较。

2.往 HashSet 中添加该对象

```java
User user1 = new User("张三",15);
User user2 = new User("张三",15);
System.out.printf("user1.equals(user2): %s, user1 hashCode: %d, user2 hashCode: %d \n",user1.equals(user2),user1.hashCode(),user2.hashCode());
HashSet<User> set = new HashSet<>();
set.add(user1);
set.add(user2);
System.out.println(set);
```

输入：

```
user1.equals(user2): true, user1 hashCode: 1936129502, user2 hashCode: 365113514 
[User [name=张三, age=15], User [name=张三, age=15]]
```

可以看到这两个对象的内容是重复的，因为往 HashSet 中添加该对象时，会计算对象的 hashCode 值来判断对象加入的位置是否相同，如果不相同，那么就认为这两个对象不相等。

所以虽然 user1 和 user2 的内容相等，但是它们的 hashCode 不相等。所以 HashSet 在添加它们时，会认为它们不相等。

3.重写 HashCode()

```java
@Override
public int hashCode() {
	final int prime = 31;
	int result = 1;
	result = prime * result + age;
	result = prime * result + ((name == null) ? 0 : name.hashCode());
	return result;
}
```

输出：

```
user1.equals(user2): true, user1 hashCode: 776315, user2 hashCode: 776315 
[User [name=张三, age=15]]
```

可以看到 user1 和 user2 的 hashCode 相等，往 HashSet 中添加它们时，首先会根据 hashCode 判断对象加入的位置是否相同，如果相同，那么就再去判断它们的内容是否一样，如果一样，那么会认为它们是相等的。

## 总结

1.对于基本类型，== 比较的是字面量

2.对于引用类型，== 比较的是地址值

3.没有重写 equals()，则比较的是地址值，相当于 == 比较

4.重写 equals()，则比较内容是否相等

5.对象相等，hashCode() 一定相等

6.hashCode() 相等，对象不一定相等

7.对象相等，equals() 一定相等

8.equals() 相等，对象不一定相等

9.equals() 和 hashCode() 没有关系

10.在本质是散列表的集合中使用某个对象，则必须重写该对象的 equals() 和 hashCode() 方法