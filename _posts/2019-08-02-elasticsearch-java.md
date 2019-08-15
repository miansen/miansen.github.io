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
    // 获取请求信息
	RequestLine requestLine = response.getRequestLine();
	// 获取响应主机信息
	HttpHost host = response.getHost();
	// 获取响应状态
	StatusLine statusLine = response.getStatusLine();
	// 获取响应头
	Header[] headers = response.getHeaders();
	// 获取响应体
	String responseBody = EntityUtils.toString(response.getEntity());
	log.debug("请求信息: " + requestLine);
	log.debug("响应主机: " + host);
	log.debug("响应状态: " + statusLine);
	for (int i = 0;i < headers.length;i++){
    	log.debug(headers[i].getName() + ": " + headers[i].getValue());
	}
	log.debug("响应体: " + responseBody);
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
        // 请求成功回调
        public void onSuccess(Response response) {
            log.debug(response);
        }
        // 请求失败回调
        public void onFailure(Exception e) {
            log.debug("failure in async scenario",e);
        }
    });
}
```

## 高级REST客户端

### 引入依赖

```xml
 <dependencies>
    <!-- 引入高级客户端 -->
    <dependency>
        <groupId>org.elasticsearch.client</groupId>
        <artifactId>elasticsearch-rest-high-level-client</artifactId>
        <version>7.2.0</version>
    </dependency>
</dependencies>
```

### 初始化

RestHighLevelClient 实例的初始化需要构建一个REST低级客户端

```java
private RestHighLevelClient client;

@Before
public void initialize() throws Exception {
    client = new RestHighLevelClient(RestClient.builder(
            new HttpHost("192.168.8.8", 9200, "http")));
}
```

### Index API

高级客户端提供两个对象创建索引：`IndexRequest`和`CreateIndexRequest`

#### IndexRequest

```java
 /**
 * 创建索引的同时插入文档，有4种方式可以构建文档，后面的文档会覆盖前面的，只能插入最后一条
 * @throws Exception
 */
@Test
public void indexRequest() throws Exception {
    // 初始化对象时指定索引名字
    IndexRequest request = new IndexRequest("roothub01");
    // 以字符串的形式构建文档，自动转换为json格式
    String jsonString = "{\"user\":\"zhangsan\",\"date\":\"2018-01-12\",\"message\":\"trying out Elasticsearch\"}";
    request.source(jsonString, XContentType.JSON);
    // 以map的形式构建文档，自动转换为json格式
    Map<String,Object> jsonMap = new HashMap<>();
    jsonMap.put("user","lisi");
    jsonMap.put("date","2019-12-12");
    jsonMap.put("message","trying out oracle");
    request.source(jsonMap);
    // 以XContentBuilder对象构建文档，由内置的帮助器生成json
    XContentBuilder builder = XContentFactory.jsonBuilder();
    builder.startObject();
    {
        builder.field("user","wangwu");
        builder.field("date","2020-12-21");
        builder.field("message","trying out Elasticsearch");
    }
    builder.endObject();
    request.source(builder);
    // 以键值对对象构建文档，它将自动转换为json格式
    request.source("user","Tom","date",new Date(),"message","trying out mysql");
    // 同步执行
    IndexResponse response = client.index(request, RequestOptions.DEFAULT);
    log.debug(response);
    // 异步执行
    client.indexAsync(request,null,new ActionListener<IndexResponse>(){
        @Override
        public void onResponse(IndexResponse indexResponse) {

        }

        @Override
        public void onFailure(Exception e) {

        }
    });
}
```

#### CreateIndexRequest

```java
/**
 * 创建索引的同时指定分片和插入文档
 * 插入文档的用法跟上面的一样
 * @throws Exception
 */
@Test
public void createIndexRequest() throws Exception {
    // 初始化对象时指定索引名字
    CreateIndexRequest request = new CreateIndexRequest("roothub");
    // 创建索引的同时可以设置分片
    request.settings(Settings.builder()
            .put("index.number_of_shards",1)
            .put("index.number_of_shards", 5));
    // 创建索引
    CreateIndexResponse response = client.indices().create(request, RequestOptions.DEFAULT);
    log.debug(response.toString());
}
```
### Get API

```java
@Test
public void getRquest() throws Exception {
    GetRequest getRequest = new GetRequest("roothub01","1");
    GetResponse getResponse = client.get(getRequest,null);
    log.debug(getResponse);
}
```