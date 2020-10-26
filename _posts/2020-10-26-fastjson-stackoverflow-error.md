---
layout: post
title: 解决 fastjson 对象转 json string 循环引用栈溢出的问题
date: 2020-10-26
categories: 杂七杂八
tags: fastjson StackOverflowError
author: 龙德
---

* content
{:toc}

对象是 1 对多，多对 1 的双向映射关系。

```java
public class Student {

    private Long id;

    private String name;

    private Classroom classroom;
}
```

```java
public class Classroom {

    private Long id;

    private List<Student> students;
}
```

![image](https://miansen.wang/assets/fastjson-stackoverflow-1.png)

报错信息如下：

```java
Exception in thread "main" java.lang.StackOverflowError
    at com.alibaba.fastjson.JSON.getMixInAnnotations(JSON.java:1383)
    at com.alibaba.fastjson.serializer.SerializeConfig.get(SerializeConfig.java:890)
    at com.alibaba.fastjson.serializer.SerializeConfig.getObjectWriter(SerializeConfig.java:446)
    at com.alibaba.fastjson.serializer.SerializeConfig.getObjectWriter(SerializeConfig.java:442)
    at com.alibaba.fastjson.serializer.JSONSerializer.getObjectWriter(JSONSerializer.java:448)
    at com.alibaba.fastjson.serializer.JSONSerializer.writeWithFieldName(JSONSerializer.java:358)
    at com.alibaba.fastjson.serializer.ASMSerializer_2_Classroom.writeDirectNonContext(Unknown Source)
    at com.alibaba.fastjson.serializer.ASMSerializer_1_Student.writeDirectNonContext(Unknown Source)
    at com.alibaba.fastjson.serializer.ASMSerializer_2_Classroom.writeDirectNonContext(Unknown Source)
    at com.alibaba.fastjson.serializer.ASMSerializer_1_Student.writeDirectNonContext(Unknown Source)
    at com.alibaba.fastjson.serializer.ASMSerializer_2_Classroom.writeDirectNonContext(Unknown Source)
    at com.alibaba.fastjson.serializer.ASMSerializer_1_Student.writeDirectNonContext(Unknown Source)
    at com.alibaba.fastjson.serializer.ASMSerializer_2_Classroom.writeDirectNonContext(Unknown Source)
    at com.alibaba.fastjson.serializer.ASMSerializer_1_Student.writeDirectNonContext(Unknown Source)
    at com.alibaba.fastjson.serializer.ASMSerializer_2_Classroom.writeDirectNonContext(Unknown Source)
```

解决方法：

1.在双向映射的一方添加 "@JSONField(serialize = false)"。

```java
public class Classroom {

    private Long id;

    @JSONField(serialize = false)
    private List<Student> students;
}
```

2.不要开启 SerializerFeature.DisableCircularReferenceDetect。