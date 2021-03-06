---
layout: post
title: 解析前后端交互的多种数据类型
date: 2020-05-01
categories: Java
tags: Content-Type
author: 龙德
---

* content
{:toc}

在 HTTP 协议的请求头中，有一个很重要的属性：Content-Type，它的作用是：

- 作为请求：告诉服务器，客户端发送的数据类型。

- 作为响应：告诉客户端，服务器响应的数据类型。

客户端或者服务器拿到数据后，通过该数据类型，就可以正确的解析数据。

由于 GET 请求不存在请求体部分，它的参数都是拼接在 URL 尾部，浏览器把数据转换成一个字符串（key1=value1&key2=value2...），然后把这个字符串追加到 URL 后面，用 ? 分割，因此 GET 请求的请求头不需要设置 Content-Type 字段。

所以下面我们所讲的都是针对 POST 请求而言。

Content-Type 属性的值有很多种，下面列举几种最常见的：

## application/x-www-form-urlencoded

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

当我们提交时，在 Chrome 浏览器中，可以看到请求体的数据格式是这样子的：

![image](/assets/20200501174443.jpg)

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

当 Spring MVC 收到客户端发送来的数据时，根据 Content-Type 声明的数据类型，通过使用 HandlerAdapter 配置的 HttpMessageConverters 来解析请求体中的数据，然后绑定到相应的 bean 上，bean 的类型、个数和顺序跟我们方法声明的参数是一样的。这样后台就接收到了数据。

![image](/assets/20200501175733.jpg)

## application/json

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

![image](/assets/20200501181634.jpg)

## multipart/form-data

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

![image](/assets/20200501184458.jpg)

首先先看 Content-Type 属性，它的值是：multipart/form-data; boundary=----WebKitFormBoundaryxJ5HRAtPAwUo1RsG

multipart/form-data 是我们在表单里指定的数据类型，这个是必须的。

那么这个 boundary=----WebKitFormBoundaryxJ5HRAtPAwUo1RsG 又是什么呢？

boundary 是边界符，作用是用来分割多个表单项和文件。它是浏览器自动帮我们加上的，边界符的值不是固定的，可以随便取，边界符也是必须的。


通过上图，我们来分析一下请求体的数据结构。

首先先看一个公式：

```
分割符 = "--" + 边界符

结束符 = "--" + 边界符 + "--"

```

根据上面的公式，我们可以得出分割符和结束符的值：

```

分割符：------WebKitFormBoundaryxJ5HRAtPAwUo1RsG

结束符：------WebKitFormBoundaryxJ5HRAtPAwUo1RsG--
```

先分析第一个表单项：

```
------WebKitFormBoundary8vr5fnZ5TkqDW6kZ
Content-Disposition: form-data; name="username"

zhangsan
```

第一行是分割符，但是仅仅有分割符是不够的，这样的数据格式还是不对，服务器还是无法解析数据。通过查资料得知，分割符必须以回车符（\r）+换行符（\n）结尾。


也就是说，第一行的内容其实是：分割符 + 回车符 + 换行符。

```
------WebKitFormBoundaryxJ5HRAtPAwUo1RsG\r\n
```

接下来看第二行，第二行的作用是告诉服务器对应字段（表单）的相关信息，第二行也必须以 "回车符 + 换行符" 结尾。所以第二行的内容其实是：


```
Content-Disposition: form-data; name="username"\r\n
```

接下来看第三行，第三行虽然没有显示任何内容，但第三行的内容其实是 "回车符 + 换行符"，只是不显示而已。

也就是说第三行的内容其实是：

```
\r\n
```

接下来看第四行，第四行是我们输入的表单值了，同样必须以 "回车符 + 换行符" 结尾。

也就是说第四行的内容其实是：

```
zhangsan\r\n
```

综上所述，一个完整的文本格式的数据结构应该是这样的：


```
------WebKitFormBoundary8vr5fnZ5TkqDW6kZ\r\nContent-Disposition: form-data; name="username"\r\n\r\nzhangsan\r\n
```


接下来再分析一下文件的内容：

```
------WebKitFormBoundary8vr5fnZ5TkqDW6kZ
Content-Disposition: form-data; name="avatar"; filename=""
Content-Type: application/octet-stream


```

文件只比文本多了一行 "Content-Type: application/octet-stream" 内容，但其实两者的数据结构其实是一样的：

```
------WebKitFormBoundary8vr5fnZ5TkqDW6kZ\r\nContent-Disposition: form-data; name="avatar"; filename=""\r\nContent-Type: application/octet-stream\r\n\r\n(文件的二进制数据)\r\n
```

最后一行是结束符，结束符也必须以 "回车符 + 换行符" 结尾。

所以最后一行的内容其实是：

```
------WebKitFormBoundary8vr5fnZ5TkqDW6kZ--\r\n
```

综上所述，请求体的完整数据结构如下图：

![image](/assets/20200502190612.jpg)