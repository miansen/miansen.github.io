---
layout: post
title: Java 并发之 synchronized
date: 2020-06-04
categories: Java
tags: 并发 synchronized
author: 龙德
---

* content
{:toc}

Java 提供了多线程之间同步的机制，其中最基本的就是使用 synchronization 关键字。这个关键字大家用得比较多，但是可能没仔细梳理过它，这篇博客将仔细梳理 synchronization 关键字，让大家对它有一个更全面和深刻的认识。

想要全面了解 synchronized，得先了解 3 个知识点：Java 内存模型（JMM）、对象的内存布局和 Monitor 对象。

## Java 内存模型（JMM）

> 现代多核 CPU 中每个核心拥有自己的一级缓存或一级缓存加上二级缓存等，问题就发生在每个核心的占缓存上。每个核心都会将自己需要的数据读到独占缓存中，数据修改后也是写入到缓存中，然后等待刷入到主存中。所以会导致有些核心读取的值是一个过期的值。
>
> Java 作为高级语言，屏蔽了这些底层细节，用 JMM 定义了一套读写内存数据的规范，虽然我们不再需要关心一级缓存和二级缓存的问题，但是，JMM 抽象了主内存和本地内存的概念。
> 
> 所有的共享变量存在于主内存中，每个线程有自己的本地内存，线程读写共享数据也是通过本地内存交换的，所以可见性问题依然是存在的。这里说的本地内存并不是真的是一块给每个线程分配的内存，而是 JMM 的一个抽象，是对于寄存器、一级缓存、二级缓存等的抽象。

![image](https://miansen.wang/assets/20200607224621.png)

总而言之，在 Java 内存模型下，所有的共享变量都存储在主内存中，每条线程还有自己的工作内存，线程的工作内存保存的是用到的共享变量的主内存副本的拷贝。线程对共享变量的所有操作都必须在工作内存中进行，而不能直接读写主内存。这就可能造成一个线程修改了一个变量的值，而另外一个线程还继续使用它在工作内存中的变量值的拷贝，造成数据的不一致。

那 JMM 跟 synchronized 有什么关系呢？

> 一个线程在获取到监视器锁以后才能进入 synchronized 控制的代码块，一旦进入代码块，首先，该线程对于共享变量的缓存就会失效，因此 synchronized 代码块中对于共享变量的读取需要从主内存中重新获取，也就能获取到最新的值。
> 
> 退出代码块的时候的，会将该线程写在缓冲区中的数据刷到主内存中，所以在 synchronized 代码块之前或 synchronized 代码块中对于共享变量的操作随着该线程退出 synchronized 块，会立即对其他线程可见（这句话的前提是其他读取共享变量的线程会从主内存读取最新值）。

也就是说，当线程进入 synchronized 控制的代码块中，这个线程对共享变量的读取从主内存中获取，退出 synchronized 块，会将修改后的数据刷到主内存中。

## 对象的内存布局

![image](https://miansen.wang/assets/Object-is-memory-layout.png)

一个 Java 对象在堆内存中包括对象头、实例数据和补齐填充 3 个部分。

- 对象头包括 Mark Words（存储哈希码、GC分代年龄、锁标志位等）和 Klass Words（指向当前对象所属的类的地址，也就是 Class 对象），如果是数组对象，还有一个保存数组长度的空间。

- 实例数据是对象真正存储的有效信息，包括了对象的所有成员变量，其大小由各个成员变量的大小共同决定。

- 对齐填充不是必然存在的，仅仅起占位符的作用。

对象的内存布局跟 synchronized 有什么关系呢？

别着急，下面介绍 synchronized 的使用的时候，会根据对象的内存布局来画图，更加直观的了解 synchronized。现在只需要记住对象头就可以了，对象头的 Mark Words 有一块空间记录了指向 Monitor 对象的指针，通过它可以关联到 Monitor 对象。

![image](https://miansen.wang/assets/Object-header.jpg)

## Monitor 对象

Monitor 其实是对 Java 对象的锁的一种抽象，称为监视器锁。每个对象都有一个 Monitor 相关联，它和 Java 对象是一对一的关系的。为了便于理解，我们可以简单的把它想象成一个对象。

Monitor 对象记录了持有锁的线程信息、阻塞队列、等待队列等。既然是对象就会有字段，Monitor 对象主要包含以下三个字段：

- _Owner：记录当前持有锁的线程
- _EntryList：阻塞队列，记录所有阻塞等待锁的线程
- _WaitSet：等待队列，记录调用 wait() 方法并还未被通知的线程

当线程获得对象的监视器锁的时候，线程 id 等信息会拷贝进 _Owner 字段，其余线程会进入阻塞队列 _Entrylist，当持有锁的线程执行 wait() 方法，会立即释放锁进入 _Waitset 队列。当线程释放锁的时候，_Owner 会被置空，然后 _Entrylist 中的线程会竞争锁，竞争成功的线程 id 会写入 _Owner，其余线程继续在 _Entrylist 中等待。

Java 对象如何跟 Monitor 关联？

前面说过，对象头的 Mark Words 有一块空间记录了指向 Monitor 对象的指针，通过它可以关联到 Monitor 对象。请看下图：

![image](https://miansen.wang/assets/Object-Monitor.png)

当锁状态变成重量级锁时，对象头有一块空间指向重量级锁的指针，通过它可以关联到 Monitor 对象。

关于锁的 3 种状态：

JDK1.6 对 synchronized 的实现引入了大量的优化，如偏向锁、轻量级锁、重量级锁等技术来减少 synchronized 操作的开销。，这 3 种锁的区别如下：

- 偏向锁是指一段同步代码一直被一个线程所访问，那么该线程会自动获取锁。降低获取锁的代价。

- 轻量级锁是指当锁是偏向锁的时候，被另一个线程所访问，偏向锁就会升级为轻量级锁，其他线程会通过自旋的形式尝试获取锁，不会阻塞，提高性能。

- 重量级锁是指当锁为轻量级锁的时候，另一个线程虽然是自旋，但自旋不会一直持续下去，当自旋一定次数的时候，还没有获取到锁，就会进入阻塞，该锁膨胀为重量级锁。重量级锁会让其他申请的线程进入阻塞，性能降低。

## synchronized 的使用

synchronized 关键字主要有两种使用方式：

- synchronized 作用于方法，称为同步方法。同步方法被调用时，会自动执行加锁操作，只有加锁成功，方法体才会得到执行。如果被 synchronized 修饰的方法是实例方法，那么这个实例的监视器会被锁定。如果是 static 方法，线程会锁住相应的 Class 对象的监视器。方法体执行完成或者异常退出后，会自动执行解锁操作。

- synchronized 代码块。synchronized(object) 在对某个对象上执行加锁时，会尝试在该对象的监视器上进行加锁操作，只有成功获取锁之后，线程才会继续往下执行。线程获取到了监视器锁后，将继续执行 synchronized 代码块中的代码，如果代码块执行完成，或者抛出了异常，线程将会自动对该对象上的监视器执行解锁操作。

### synchronized 修饰实例方法

synchronized 修饰实例方法，锁住的是实例对象的监视器，请看下面的例子。

```java
public class Person {

	private String name;
	
	private Integer age;
	
	private Dog dog;

	public Person(String name, Integer age, Dog dog) {
		this.name = name;
		this.age = age;
		this.dog = dog;
	}

	public synchronized void setA() {
		System.out.println(Thread.currentThread().getName() + " 线程进入了 setA() 方法");
	}
	
	public synchronized void setB() {
		System.out.println(Thread.currentThread().getName() + " 线程进入了 setB() 方法");
	}
	
	// 省略 Getters 和 Setters...
}
```

```java
public class Dog {

	private String name;
	
	private Integer age;

	public Dog(String name, Integer age) {
		this.name = name;
		this.age = age;
	}
	
	// 省略 Getters 和 Setters...
}
```

```java
public static void main(String[] args) {
	Dog dog = new Dog("汪汪", 1);
	Person zhangsan = new Person("张三", 15, dog);
	Thread t1 = new Thread(() -> zhangsan.setA());
	Thread t2 = new Thread(() -> zhangsan.setA());
	t1.start();
	t2.start();
}
```

启动 2 个线程，共同访问张三的 setA() 方法，由于 setA() 方法被 synchronized 修饰，所以只会有一个线程获得张三的监视器锁，获得锁之后才有资格执行 setA() 方法，其它获取不到锁的线程都会阻塞。

![image](https://miansen.wang/assets/20200606180550.jpg)

如果 t2 线程访问的是 setB() 方法呢？

```java
Thread t2 = new Thread(() -> zhangsan.setB());
```

也会发生多个线程争同一把锁的问题，因为 setA() 和 setB() 都是实例方法，synchronized 锁住的都是张三这个实例的监视器。

![image](https://miansen.wang/assets/20200606181541.jpg)

这时候的内存结构是这样的：

![image](https://miansen.wang/assets/20200606182510.png)

我画的不是很严谨，仅供参考，大家能理解就好。

如果 t2 线程访问的是李四的 setA() 方法呢？

```java
Person lisi = new Person("李四", 16, dog);
Thread t2 = new Thread(() -> lisi.setA());
```

这时候这两个线程都能执行各自对象的 setA() 方法，因为 t1 线程竞争的是张三的监视器锁，t2 线程竞争的是李四的监视器锁。这是两个不同的锁了，自然不会发生多个线程争同一把锁的问题。

![image](https://miansen.wang/assets/20200606183227.jpg)

但是这样是有安全隐患的，别忘了张三和李四养的都是同一条狗，如果在 setA() 方法里对这条狗做了增删改操作，那么势必会引起线程安全问题。如果是这种情况，那么必须在狗对象里思考可能引起的线程安全问题，并作出相应的同步操作。

这时候的内存结构大概是这样的：

![image](https://miansen.wang/assets/20200606190817.png)

### synchronized 修饰静态方法

Person 类新增两个静态方法

```java
public synchronized static void setC() {
	System.out.println(Thread.currentThread().getName() + " 线程进入了 setC() 方法");
}

public synchronized static void setD() {
	System.out.println(Thread.currentThread().getName() + " 线程进入了 setD() 方法");
}
```

t1 和 t2 线程都访问 setC() 方法

```java
Thread t1 = new Thread(() -> zhangsan.setC());
Thread t2 = new Thread(() -> zhangsan.setC());
```

前面说过，synchronized 修饰静态方法，锁住的是相应的 Class 对象的监视器。Class 对象是啥对象，干啥用的？怎么生成的？下面我就简单的说一下。

![image](https://miansen.wang/assets/20200606192158.jpg)

个人觉得学习 Java 有几座难以跨越的大山，除了并发这座大山之外，Class 对象也算得上一座大山了。Class 对象其实不是很好理解，只有理解了 Class 对象，才能理解类加载机制、类型机制、反射以及 synchronized 修饰静态方法等等重要的知识点。

简单的来说，Class 对象也就是 Class 类的对象，我们每写的一个 Java 类，都会编译成一个 .class 文件，这个文件包含了类的字段、方法、注解等等元信息。既然在 Java 里万物都是对象，那么 .class 文件就可以看成一个个的对象，哪个类的对象呢？就是 Class 类的对象。JVM 加载 .class 文件的时候，会在方法区生成一个个 Class 对象。然后我们每 new 一个实例对象，其实都是根据这个类对应的 Class 对象 new 出来的实例对象。不管你 new 多少个，这些实例对象都是根据 Class 对象这个模板生成的，这些实例对象都指向同一个 Class 对象。前面说对象头的时候，有个 Klass Words 空间，里面存的就是指向 Class 对象的指针，通过它就可以关联到 Class 对象。

![image](https://miansen.wang/assets/20200607212042.png)

所以 t1 和 t2 线程都访问 setC() 方法，由于 setC() 方法是静态方法，并且被 synchronized 修饰，锁的是张三对应的 Class 对象的监视器，所以会发生多个线程争同一把锁的问题。

![image](https://miansen.wang/assets/20200606194236.jpg)

如果 t2 线程访问的是 setD() 方法呢？

```java
Thread t2 = new Thread(() -> zhangsan.setD());
```
答案大家都想到了，也会发生多个线程争同一把锁的问题。因为 setC() 和 setD() 都是静态方法，synchronized 锁住的都是张三这个实例对应的 Class 对象的监视器，所以会发生多个线程争同一把锁的问题。

![image](https://miansen.wang/assets/20200606194915.jpg)


如果 t2 线程访问的是 setA() 方法呢？

那这时候不会发生多个线程争同一把锁的问题。因为 t1 线程竞争的是张三的 Class 对象的监视器锁，t2 线程竞争的是张三这个实例对象的监视器锁。这是两个不同的锁了，自然不会发生多个线程争同一把锁的问题。

![image](https://miansen.wang/assets/20200606195751.jpg)

如果 t2 线程访问的是李四的 setC() 方法呢？

```java
Thread t2 = new Thread(() -> lisi.setC());
```

由于张三和李四的 Class 对象都是同一个，所以张三和李四的 setC() 方法锁的都是同一个 Class 对象的监视器，所以也会发生多个线程争同一把锁的问题。

![image](https://miansen.wang/assets/20200607213318.jpg)

这时候的内存结构图大概是这样的：

![image](https://miansen.wang/assets/20200607222238.png)

### synchronized 代码块

Person 类新增 setE() 方法

```java
public void setE() {
	synchronized (dog) {
		System.out.println(Thread.currentThread().getName() + " 线程进入了同步代码块");
	}
}
```

可以看到 setE() 方法有个同步代码块，锁住的是 dog 对象的监视器。

t1 和 t2 线程都访问张三的 setE() 方法

```java
Thread t1 = new Thread(() -> zhangsan.setE());
Thread t2 = new Thread(() -> zhangsan.setE());
``` 

结果大家肯定想到了，肯定会发生多个线程争同一把锁的问题。

![image](https://miansen.wang/assets/20200606200543.jpg)

如果 t2 线程访问的是李四的 setE() 方法呢？

```java
Thread t2 = new Thread(() -> lisi.setE());
```

也会发生多个线程争同一把锁的问题。因为张三和李四养的都是同一条狗。

![image](https://miansen.wang/assets/20200606200944.jpg)

这时候的内存结构图大概是这样的：

![image](https://miansen.wang/assets/20200607223028.png)

如果李四养的是另一条狗呢？

```java
Dog haha = new Dog("哈哈", 2);
Person lisi = new Person("李四", 16, haha);
Thread t2 = new Thread(() -> lisi.setE());
```

这时候就不会发生多个线程争同一把锁的问题。因为张三和李四的狗不是同一只，这两个线程抢的不是同一把锁。

![image](https://miansen.wang/assets/20200606201323.jpg)

这时候的内存结构图大概是这样的：

![image](https://miansen.wang/assets/20200607223937.png)

## 总结

总的来说就两点：

1.为什么需要 synchronized

因为在 Java 内存模型下，所有的共享变量都存储在主内存中，每条线程还有自己的工作内存，线程的工作内存保存的是用到的共享变量的主内存副本的拷贝。线程对共享变量的所有操作都必须在工作内存中进行，而不能直接读写主内存。这就可能造成一个线程修改了一个变量的值，而另外一个线程还继续使用它在工作内存中的变量值的拷贝，造成数据的不一致。所以在多线程的环境中，必须要考虑多个线程访问临界资源的安全问题。可以使用 Java 语言提供的 synchronized 关键字或者 java.util.concurrent.locks 包下的各种锁（比如 ReentrantLock） 来保证线程操作的原子性。

对于 synchronized 来说，当线程进入 synchronized 控制的代码块中，这个线程对共享变量的读取从主内存中获取，退出 synchronized 块，会将修改后的数据刷到主内存中。

2.如何使用 synchronized

要分清楚 synchronized 锁的是哪个对象的监视器：

- synchronized 修饰实例方法，那么这个实例对象的监视器会被锁定。

- synchronized 修饰静态方法，那么这个实例对象对应的 Class 对象的监视器会被锁定。

- synchronized 作用于代码块，那么括号里的对象的监视器会被锁定。

## 参考资料

[https://javadoop.com/post/java-memory-model](https://javadoop.com/post/java-memory-model)

[https://www.cnblogs.com/ZoHy/p/11313155.html](https://www.cnblogs.com/ZoHy/p/11313155.html)

[https://zhuanlan.zhihu.com/p/138427106](https://zhuanlan.zhihu.com/p/138427106)