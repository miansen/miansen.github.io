---
layout: post
title: Elasticsearch7.x 入门
date: 2019-07-20
categories: Elasticsearch
tags: Elasticsearch
author: 龙德
---

* content
{:toc}

Elasticsearch 是一个 nosql 数据库，比传统的关系型数据库厉害的地方在于全文搜索和分析数据。

它底层是开源库 Lucene，Elasticsearch 是 Lucene 的封装，提供了 RESTful 风格的 API，开箱即用。




## 安装

我的环境：

- CentOS-6.5-i386 32位
- JDK 1.8

官方下载地址：[https://www.elastic.co/cn/downloads/elasticsearch](https://www.elastic.co/cn/downloads/elasticsearch)

我这里下载的是最新版本

```shell
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.2.0-linux-x86_64.tar.gz
```

我直接解压在当前目录了

```shell
tar zxvf elasticsearch-7.2.0-linux-x86_64.tar.gz -C
```

## 运行

输入以下命令运行 elasticsearch

```shell
cd elasticsearch-7.2.0
./bin/elasticsearch
```

如果这时候报错：

```
lasticsearchException[X-Pack is not supported and Machine Learning is not available for [linux-x86]; you can use the other X-Pack features (unsupported) by setting xpack.ml.enabled: false in elasticsearch.yml]
```

则编辑 elasticsearch.yml 文件

```shell
vi config/elasticsearch.yml
```

在最末尾加入以下配置

```yml
xpack.ml.enabled: false
```

保存退出，然后再启动

## 访问

如果启动成功，当前窗口会一直堵塞着，这时候打开另一个窗口访问`curl http://localhost:9200?pretty`，会输出以下信息：

```json
[sen@localhost elasticsearch-7.2.0]$ curl http://localhost:9200?pretty
{
  "name" : "localhost.localdomain",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "bdHwweGNTCSXLCnI8txzkQ",
  "version" : {
    "number" : "7.2.0",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "508c38a",
    "build_date" : "2019-06-20T15:54:18.811730Z",
    "build_snapshot" : false,
    "lucene_version" : "8.0.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

输出这些信息就说明 elasticsearch 在 9200 端口成功运行了

## 基本概念

- **索引（Index）**

这里的索引是一个名词，索引是 elasticsearch 存储数据的顶层单位，对应的是关系型数据库中的数据库，索引的名字必须是小写。

- **类型（Type）**

类型对应的是关系型数据库中的表

> 在Elasticsearch 7.0.0或更高版本中创建的索引不再接受 _default_映射。在6.x中创建的索引将继续像以前一样在Elasticsearch 
> 6.x中运行。在7.0中的API中不推荐使用类型，对索引创建，放置映射，获取映射，放置模板，获取模板和获取字段映射API进行重大更改。

类型会在8.X版本彻底移除

下面的演示我还是保留了类型这个概念

- **文档（Document）**

Index 里面单条的记录称为 Document（文档），文档是一个可被索引的基础信息单元。

文档以 JSON 格式来表示。

文档对应的是关系型数据库中的行。

- **字段（Field）**

文档的 key，对应的是关系型数据库中的列

和关系型数据库的比较：

```
Relational DB -> Databases -> Tables -> Rows -> Columns
Elasticsearch -> Indices   -> Types  -> Documents -> Fields
```

## 创建索引

文档的存储单元是索引，好比我们往关系型数据库里插入一条数据之前，得先把数据库创建出来。

所以我们先创建一个索引，elasticsearch 的 API 是基于 TCP/IP 的，所以我们可以发送 PUT 请求创建索引

```shell
curl -X PUT 'http://localhost:9200/customer'
```

响应结果：

```json
{"acknowledged":true,"shards_acknowledged":true,"index":"customer"}
```

可以看到成功创建了 customer 索引

## 删除索引

发起 delete 请求删除索引

```shell
curl -X DELETE 'http://localhost:9200/customer'
```

响应结果：

```json
{"acknowledged":true}
```

## 创建文档

其实应该叫 **"索引文档"** 更准确，这里的索引是动词，"索引文档" 表示把一个文档存储到索引（名词）里，就像 SQL 中的 INSERT 关键字，差别是，如果文档已经存在，新的文档将覆盖旧的文档

向指定的 index/type/id 发送 put 请求，就可以索引文档

```shell
curl -X PUT 'http://localhost:9200/customer/user/1' -H 'Content-Type:application/json' -d '{"username":"zhangsan","age":14}'
```

因为 ealsticsearch 的 http 接口是 RESTful 风格的，所以从上面的 put 请求我们可以得出以下信息：

|名字|说明|
|---:|---:|
|customer|索引名|
|user|类型名|
|1|这个员工的ID|
|username|这个员工的名字|
|age|这个员工的年龄|

响应结果：

```json
{"_index":"customer","_type":"user","_id":"1","_version":1,"result":"created","_shards":{"total":2,"successful":1,"failed":0},"_seq_no":0,"_primary_term":1}
```

索引文档之前，如果索引和类型不存在，那么 ealsticsearch 会自动创建该索引和类型

PUT 请求在 RESTful 里对应的 CURD 操作是更新，如果在 elasticsearch 里存在同一个索引的同一个文档，那么上面那条请求就变成了更新

我们可以再次发送一下上面的 PUT 请求，不过 username 的值换成 lisi

```shell
curl -X PUT 'http://localhost:9200/customer/user/1' -H 'Content-Type:application/json' -d '{"username":"lisi","age":14}'
```

响应结果：

```json
{"_index":"customer","_type":"user","_id":"1","_version":2,"result":"updated","_shards":{"total":2,"successful":1,"failed":0},"_seq_no":1,"_primary_term":1}
```

注意看 "_version" 字段，它的值变成了 "2"，"result" 字段，它的值变成了 "updated"

elasticsearch 每条文档都有一个 _version 字段，记录着这条文档的版本，每当文档更新时，版本都会加 1，所以我们这条 PUT 请求就变成了更新

要想发起真正的索引请求，可以用 POST，不过不要指定 id

```shell
curl -X POST 'http://localhost:9200/customer/user' -H 'Content-Type:application/json' -d '{"username":"haha","age":14}'
```

响应结果：

```json
{"_index":"customer","_type":"user","_id":"ICGAEWwB7frJTf7vsLAQ","_version":1,"result":"created","_shards":{"total":2,"successful":1,"failed":0},"_seq_no":3,"_primary_term":1}
```

可以看到 "result" 字段的值是 "created"，"_id" 字段变成了 elasticsearch 自动生成的随机字符串，用的是 UUIDS。

POST 请求其实也可以指定 id 的，不过如果指定的 id 已存在的话，POST 请求还是会变成更新。

所以说到底，PUT 和 POST 请求差别不大，用 PUT，如果文档不存在，则会变成新增；用 POST，如果文档已存在，则会变成更新。

## 删除文档

发起 delete 请求删除文档

```shell
curl -X DELETE 'http://localhost:9200/customer/user/1'
```

响应结果：

```json
{"_index":"customer","_type":"user","_id":"1","_version":2,"result":"deleted","_shards":{"total":2,"successful":1,"failed":0},"_seq_no":2,"_primary_term":1}
```

注意看 "_version" 变成了 "2"

如果文档未找到，我们将得到一个404 Not Found状态码，响应体是这样的：

```json
{
  "_index" : "customer",
  "_type" : "user",
  "_id" : "1",
  "_version" : 1,
  "result" : "not_found",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 3,
  "_primary_term" : 1
}
```