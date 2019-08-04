---
layout: post
title: Elasticsearch-Java客户端
date: 2019-08-02
categories: Elasticsearch
tags: Elasticsearch
author: 龙德
excerpt: Elasticsearch 官方提供了3种客户端：1.TransportClient，这个客户端在7.0.0弃用，将在8.0版本中删除。2.elasticsearch-rest-client，低级的REST客户端，基于Apache HTTP，支持所有Elasticsearch版本，请求编码和响应解码留给用户实现（自己写JSON）。3.elasticsearch-rest-high-level-client，高级的REST客户端，基于低级客户端，对增删改查进行了封装，提供特定方法的API，不需要用户处理编解码，类似之前的TransportClient，但是兼容性较差，对客户端和集群版本要求较高。
---

* content
{:toc}

Elasticsearch 官方提供了3种客户端：

1.TransportClient，这个客户端在7.0.0弃用，将在8.0版本中删除。

2.elasticsearch-rest-client，低级的REST客户端，基于Apache HTTP，支持所有Elasticsearch版本，请求编码和响应解码留给用户实现（自己写JSON）。

3.elasticsearch-rest-high-level-client，高级的REST客户端，基于低级客户端，对增删改查进行了封装，提供特定方法的API，不需要用户处理编解码，类似之前的TransportClient，但是兼容性较差，对客户端和集群版本要求较高。

TransportClient 这个客户端因为被废弃所以就不做演示了，下面演示低级和高级的REST客户端。

## 低级REST客户端

### 引入依赖

```xml
<!-- 引入低级客户端 -->
<dependency>
    <groupId>org.elasticsearch.client</groupId>
    <artifactId>elasticsearch-rest-client</artifactId>
    <version>7.2.0</version>
</dependency>
<!--引入单元测试-->
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.12</version>
    <scope>test</scope>
</dependency>
<!--引入日志-->
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-core</artifactId>
    <version>2.11.1</version>
</dependency>
```

### 初始化

```java
private RestClient restClient;

@Before
public void initialize() throws Exception {
    // builder的参数可以是多个Host数组
    restClient = RestClient.builder(new HttpHost("192.168.8.8", 9200, "http")).build();
}
```

### 发送请求

一旦创建了`RestClient`，就可以通过调用其中一个`performRequest`或`performRequestAsync`方法来发送请求。

`performRequest`方法是同步的，直接返回`Response`，这意味着客户端将阻塞并等待返回的响应。

`performRequestAsync`方法是异步的，返回`void`，接受一个额外的`ResponseListener`作为参数，监听请求完成或失败，并作出相应的处理。

#### 发送同步请求

```java
@Test
public void performRequest() throws Exception {
    // 构建请求方法和请求路径
    Request request = new Request("GET", "/customer/_doc/1");
    // 添加请求参数
    request.addParameter("pretty", "true");
    // 添加请求体，为 HttpEntity 指定的 ContentType 非常重要，因为它将用于设置 content-type 头部，以便 Elasticsearch 能够正确解析内容。
    request.setEntity(new NStringEntity("{\"query\":{\"match\":{\"name\":\"zhangsan\"}}}",
            ContentType.APPLICATION_JSON));
    // 发送请求，直接返回了响应对象
    Response response = restClient.performRequest(request);
}
```

#### 发送异步请求

```java
@Test
public void performRequestAsync() throws Exception {
    // 构建请求方法和请求路径
    Request request = new Request("GET", "/customer/_doc/1");
    // 添加请求参数
    request.addParameter("pretty", "true");
    // 添加请求体
    request.setEntity(new NStringEntity("{\"query\":{\"match\":{\"name\":\"zhangsan\"}}}",
            ContentType.APPLICATION_JSON));
    // 发送异步请求
    restClient.performRequestAsync(request,new ResponseListener(){
        // 处理返回响应，响应对象作为参数
        public void onSuccess(Response response) {
            log.debug(response);
        }
        //处理失败响应
        public void onFailure(Exception e) {
            log.debug("failure in async scenario");
        }
    });
}
```

### 读取响应

同步请求和异步请求都可以拿到响应对象`Response`，我们可以从响应对象中拿到请求信息、主机信息、响应状态和响应体等信息。

```java
// 读取请求信息
RequestLine requestLine = response.getRequestLine();
// 读取响应主机信息
HttpHost host = response.getHost();
// 读取响应状态
StatusLine statusLine = response.getStatusLine();
// 读取响应头
Header[] headers = response.getHeaders();
// 读取响应体
String responseBody = EntityUtils.toString(response.getEntity());
log.debug("请求信息: " + requestLine);
log.debug("响应主机: " + host);
log.debug("响应状态: " + statusLine);
for (int i = 0;i < headers.length;i++){
    log.debug(headers[i].getName() + ": " + headers[i].getValue());
}
log.debug("响应体: " + responseBody);
```
