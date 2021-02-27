---
layout: post
title: 异常没有打印栈信息
date: 2021-02-27
categories: Java
tags: 异常
---

* content
{:toc}

查看线上日志，遇到一个诡异的问题，就是系统大量空指针的异常，但是没有打印异常栈，导致不方便定位问题。

打印日志的代码如下：

![image](https://miansen.wang/assets/20210224212425.png)

服务器的日志如下：

![image](https://miansen.wang/assets/20210224212602.png)

可以看到只打印了 `java.lang.NullpointerException`，并没有打印异常栈，这样很难定位到问题。

没有遇到过这种情况，思考了很久，也调试了代码，确定不是程序代码的问题。不知道咋解决，还好群里的大神给出了线索：

> JVM 虚拟机会对异常信息进行优化，当相同异常出现很多次，会认为它是热点异常，忽略掉异常堆栈信息；通过增加虚拟机参数：-XX: -OmitStackTraceInFastThrow 可解决。

这是 HotSpot VM 专门针对异常做的一个优化，称为 fast throw，当一些异常在代码里某个特定位置被抛出很多次的话，HotSpot Server Compiler（C2）会用 fast throw 来优化这个抛出异常的地方，直接抛出一个事先分配好的、类型匹配的对象，这个对象的 message 和 stack trace 都被清空。抛出这个异常非常快，不用额外分配内存，也不用爬栈。
副作用：正好是需要知道哪里出问题的时候看不到 stack trace 了，不利于排查问题。可以考虑通过 -XX:-OmitStackTraceInFastThrow 禁用该默认的优化。

知道原因后，我们的解决办法可以有：

1. 历史数据还在的话，下载历史数据查看。
2. 重新启动服务器，再观察日志。
3. 设置JVM参数，暂时禁止掉这个优化选项：-XX:+OmitStackTraceInFastThrow。