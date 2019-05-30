---
layout: post
title: mysql group_concat函数和oracle listagg函数的作用和区别
date: 2019-01-23
categories: MySQL Oracle
tags: group_concat listagg
author: 龙德
---

* content
{:toc}

## mysql group_concat的作用和语法

- 作用：将组中的字符串连接成为具有各种选项的单个字符串

- 语法：group_concat([DISTINCT] 要连接的字段 [Order BY ASC/DESC 排序字段] [Separator ‘分隔符’])

分隔符默认是逗号

比如有这样一个表

![image](/assets/20190123101642.jpg)

有这样一个需求：将分数用分隔符'-'以降序的方式连接在一起

我们可以这样写

`SELECT GROUP_CONCAT(A.SCORE ORDER BY A.SCORE DESC SEPARATOR '-') FROM GRADE A;`

结果如下

![image](/assets/20190123102435.jpg)




上面那个例子只是最简单的用法，GROUP_CONCAT函数通常和GROUP BY函数搭配使用

比如现在有一个需求，查询出各个学生的分数，将分数以升序的方式连在一起

我们可以这样写

`SELECT GROUP_CONCAT(A.SCORE ORDER BY A.SCORE),A.NAME FROM GRADE A GROUP BY A.NAME;`

结果如下

![image](/assets/20190123104038.jpg)

## oracle listagg的作用和语法

- 作用：将多行结果整合成一个列

- 语法：LISTAGG(要连接的字段,分隔符) WITHIN GROUP( ORDER BY 排序字段)

现在我们用oracle实现以上两种需求

第一个需求可以这样写：

`select listagg(a.score,',') within group ( order by a.score desc) from grade a;`

结果如下

![image](/assets/20190123203805.jpg)

第二个需求可以这样写：

`select listagg(a.score,',') within group ( order by a.score),a.name from grade a group by a.name;`

结果如下

![image](/assets/20190123203905.jpg)