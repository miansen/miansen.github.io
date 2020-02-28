---
layout: post
title: SpringBoot解决form表单提交中文乱码
date: 2020-02-28
categories: SpringBoot
tags: 乱码
author: 龙德
---

* content
{:toc}

form 表单提交数据映射到后台对象，中文出现乱码

![image](https://miansen.wang/assets/20200228215132.jpg)

解决方法如下：

1.application.properties 文件配置字符编码

```properties
spring.banner.charset=UTF-8
spring.messages.encoding=UTF-8
spring.http.encoding.charset=UTF-8
spring.http.encoding.force=true
spring.http.encoding.enabled=true
server.tomcat.uri-encoding=UTF-8
```

2.配置 CharacterEncodingFilter

之前 SpringMVC 是在 web.xml 里配置的

```xml
<filter>
	<description>字符集过滤器</description>
    <filter-name>encodingFilter</filter-name>
    <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
    <init-param>
    	<description>字符集编码</description>
        <param-name>encoding</param-name>
        <param-value>UTF-8</param-value>
    </init-param>
    <init-param>
		<param-name>forceEncoding</param-name>
		<param-value>true</param-value>
	</init-param>
</filter>
<filter-mapping>
    <filter-name>encodingFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```

SpringBoot 的话直接注入 bean 即可

```java
@Configuration
public class CharacterEncodingFilterConfig {

	@Bean
    public FilterRegistrationBean filterRegistrationBean() {
        FilterRegistrationBean registrationBean = new FilterRegistrationBean();
        CharacterEncodingFilter characterEncodingFilter = new CharacterEncodingFilter();
        characterEncodingFilter.setForceEncoding(true);
        characterEncodingFilter.setEncoding("UTF-8");
        registrationBean.setFilter(characterEncodingFilter);
        return registrationBean;
    }

}
```

重启之后，重新提交表单，中文就可以正常显示了

![image](https://miansen.wang/assets/20200228215924.jpg)