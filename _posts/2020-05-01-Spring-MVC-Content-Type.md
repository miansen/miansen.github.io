---
layout: post
title: Spring MVC 前后端交互的多种数据格式，从 Content-Type 说起
date: 2020-04-27
categories: SpringMVC
tags: SpringMVC Content-Type
author: 龙德
---

* content
{:toc}

在 HTTP 协议的请求头中，有一个很重要的属性：Content-Type，它的作用是：

- 作为请求：该属性的作用是：告诉服务器，客户端发送的数据类型。

- 作为响应：该属性的作用是：告诉客户端，服务器响应的数据类型。

客户端或者服务器拿到数据后，通过该数据类型，就可以正确的解析数据。

Content-Type 属性的值有很多种，下面列举几种最常见的：

- application/x-www-form-urlencoded

这是最常见的数据类型，前端表单默认使用的就是这种类型。

比如有这么一个登录表单：

```html
<form action="login.do" enctype="application/x-www-form-urlencoded" method="post">
    <input type="text" name="username" />
    <input type="password" name="password" />
    <button type="submit">登录</button>
</form>
```

注意 "enctype" 属性，该属性规定在发送到服务器之前，浏览器应该如何对表单数据进行编码。默认是 "application/x-www-form-urlencoded"。

当我们点击 "登录" 按钮时，浏览器会将所有内容进行编码，拼接成 "key=value" 的格式，也就是键值对的格式。多个键值对之间用 "&" 分割。

在 Chrome 浏览器中，可以看到请求体的数据格式是这样子的：

![image](https://miansen.wang/assets/20200501174443.jpg)

可以看到浏览器帮我们拼接成了键值对的格式，key 就是输入框的 name，而 value 就是我们输入的内容。

当服务器接收到数据后，根据 Content-Type 属性获取到数据类型，就能正确的解析数据。

接下来是后台代码：

```java
@RequestMapping(value = "/login.do", method = RequestMethod.POST)
public String login(@RequestParam(value = "username") String username,
					@RequestParam(value = "password") String password) {
	return "index.jsp";
}
```

各个注解的作用这里就不讲了，不熟悉的话可以去看这篇博客：[Spring MVC 常用的注解](https://miansen.wang/2019/06/24/4)

当 Spring MVC 收到客户端发送来的数据时，根据 Content-Type 声明的数据类型，通过使用 HandlerAdapter 配置的 HttpMessageConverters 来解析请求体中的数据，然后绑定到相应的 bean 上，bean 的类型、个数和顺序跟我们方法声明的参数是一样的。

![image](https://miansen.wang/assets/20200501175733.jpg)

- application/json

这也是一种很常见的数据类型，这种类型的数据是序列化后的 JSON 字符串。

比如有这么一个登录表单：

```html
<form>
    <input type="text" name="username" />
    <input type="password" name="password" />
    <button type="button">登录</button>
</form>
```

后台代码如下：

```java
@RequestMapping(value = "/login.do", method = RequestMethod.POST)
@ResponseBody
public String login(@RequestBody User user) {
	return "login success.";
}
```

处理 application/json 编码的数据时，必须使用 @RequestBody 注解，Spring MVC 通过使用 HandlerAdapter 配置的 HttpMessageConverters 来解析请求体中的数据，然后绑定到 @RequestBody 注解的对象上。

AJAX 代码如下：

```javascript
var username = $("input[name='username']").val();
var password = $("input[name='password']").val();
var data = {
    username: username,
    password: password
}
$.ajax({
  type: "post",
  url: "/login.do",
  contentType: "application/json",
  data: JSON.stringify(data),
  success: function(data){
    
  },
  error: function(data){

  }
});
```

JQuery 的 AJAX 默认传递的是 application/x-www-form-urlencoded 类型的数据，如果想传递 JSON 类型，则需指定 contentType: application/json。

由于 application/json 类型的数据是序列化后的 JSON 字符串，所以我们也必须手动把 JSON 对象序列化成字符串，我们可通过 JSON.stringify(data) 方法进行序列化。

通过 Chrome 浏览器可以看到请求体的数据格式是这样子的：

![image](https://miansen.wang/assets/20200501181634.jpg)

- multipart/form-data

在最初的 http 协议中，没有上传文件方面的功能。后来为了支持文件上传，提高二进制文件的传输效率，新增了 multipart/form-data 数据类型。

比如有这么一个登录表单：

```html
<form action="login.do" enctype="multipart/form-data" method="post">
    <input type="text" name="username" />
    <input type="password" name="password" />
    <input type="file" name="avatar">
    <button type="submit">登录</button>
</form>
```

表单的 enctype 属性必须指定为 multipart/form-data，否则不能上传文件。

后台代码如下：

```java
@RequestMapping(value = "/login.do", method = RequestMethod.POST)
public String login(String username, String password, MultipartFile avatar) {
	return "index.jsp";
}
```

MultipartFile 这个类是用来接受前台传过来的文件。

重点是 multipart/form-data 数据类型的请求体，这种数据类型的请求体跟前面两个完全不一样。

通过 Chrome 浏览器我们可以看到请求体的数据格式，请看下图：

![image](https://miansen.wang/assets/20200501184458.jpg)

首先先看 Content-Type 属性，它的值是：multipart/form-data; boundary=----WebKitFormBoundaryxJ5HRAtPAwUo1RsG

multipart/form-data 是我们在表单里指定的数据类型，这个是必须的。

那么这个 boundary=----WebKitFormBoundaryxJ5HRAtPAwUo1RsG 又是什么呢？

boundary 是边界符，作用是用来分割多个表单项和文件。它是浏览器自动帮我们加上的，边界符的值不是固定的，可以随便取。


通过上图，我们来分析一下请求体的数据结构。

首先先看一个公式：

```
分割符 = "--" + 边界符

结束符 = "--" + 边界符 + "--"

```

由此我们可以得出分割符和结束符的值：

```

分割符：------WebKitFormBoundaryxJ5HRAtPAwUo1RsG

结束符：------WebKitFormBoundaryxJ5HRAtPAwUo1RsG--
```

通过上图我们知道，请求体中的第一行是分割符，也就是 ------WebKitFormBoundaryxJ5HRAtPAwUo1RsG。

但是仅仅有分割符是不对的，这样服务器还是无法解析数据。通过查资料得知，分割符必须以回车符（\r）+换行符（\n）结尾。


也就是说，第一行的内容其实是：分割符 + 回车符 + 换行符。

```
------WebKitFormBoundaryxJ5HRAtPAwUo1RsG\r\n
```

接下来看第二行：

```
Content-Disposition: form-data; name="username"
```

第二行的作用是给出对应字段（表单）的相关信息，以 "回车符 + 换行符 + 回车符 + 换行符" 结尾。

也就是说第二行的内容其实是：

```
Content-Disposition: form-data; name="username"\r\n\r\n
```

接下来看第三行：

第三行的内容就是表单的值了，以 "回车符 + 换行符" 结尾。

也就是说第三行的内容其实是：

```
zhangsan\r\n
```

这样一个表单项的数据结构就分析完了，多个表单项也是以此类推。