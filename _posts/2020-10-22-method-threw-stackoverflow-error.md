---
layout: post
title: idea debug 报 Method threw 'java.lang.StackOverflowError' exception. Cannot ...
date: 2020-10-22
categories: 杂七杂八
tags: StackOverflowError
author: 龙德
---

* content
{:toc}

今天用 idea debug 的时候，遇到一个错误：Method threw 'java.lang.StackOverflowError' exception. Cannot ...

如图所示：

![image](https://miansen.wang/assets/method-threw-stackoverflow-error-1.png)

类结构如下：

```java
public class Student {

    private Long id;

    private String name;

    private Classroom classroom;
    
    @Override
    public String toString() {
        return "Student{" +
                "id=" + id +
                ", name='" + name + '\'' +
                ", classroom=" + classroom +
                '}';
    }
}
```

```java
public class Classroom {

    private Long id;

    private List<Student> students;
    
    @Override
    public String toString() {
        return "Classroom{" +
                "id=" + id +
                ", students=" + students +
                '}';
    }
}
```

因为 debug 模式下需要显示变量信息，这个信息就是要调用 toString() 方法得到的。而我采用的结构是 1 对多的关系，对象互相引用，调用 toString() 方法的时候栈溢出了。

解决这个问题也很简单，修改任意一个类的 toString() 方法，不要造成循环调用就行。

```java
public class Classroom {

    private Long id;

    private List<Student> students;

    @Override
    public String toString() {
        return "Classroom{" +
                "id=" + id +
                '}';
    }
    
}
```

![image](https://miansen.wang/assets/method-threw-stackoverflow-error-2.png)