---
layout: post
title: Elasticsearch-入门
date: 2019-07-20
categories: Elasticsearch
tags: Elasticsearch
author: 龙德
---

* content
{:toc}

Elasticsearch 是一个 nosql 数据库，它底层基于开源库 Lucene，Elasticsearch 是 Lucene 的封装，隐藏了复杂的操作，提供了 RESTful 风格的 API 和 Java 客户端（其它语言的客户端也有），开箱即用。




## 安装

我的环境：

- CentOS-6.5-i386 32位
- JDK 1.8

官方下载地址：[https://www.elastic.co/cn/downloads/elasticsearch](https://www.elastic.co/cn/downloads/elasticsearch)

我这里下载的是最新版本 7.2.0

```shell
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.2.0-linux-x86_64.tar.gz
```

我直接解压在当前目录了

```shell
tar zxvf elasticsearch-7.2.0-linux-x86_64.tar.gz -C
```

## 启动

输入以下命令启动 elasticsearch

```shell
cd elasticsearch-7.2.0
./bin/elasticsearch
```

如果报了下面这个错误：

```
lasticsearchException[X-Pack is not supported and Machine Learning is not available for [linux-x86]; you can use the other X-Pack features (unsupported) by setting xpack.ml.enabled: false in elasticsearch.yml]
```

可以编辑 config/elasticsearch.yml 文件

```shell
vi config/elasticsearch.yml
```

在最末尾加入这个配置

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

输出这些信息就说明 elasticsearch 在 9200 端口成功启动了。

### 外网访问

默认情况下，Elasticsearch 只允许本机访问，如果要开放外网访问，可以修改 config/elasticsearch.yml 文件，需要修改**3个**地方。

（1）将 **node.name: node-1** 的注释放开

（2）将 **network.host** 的注释放开，将它的值改为 0.0.0.0

`network.host: 0.0.0.0`

设成 0.0.0.0 是不限制任何 IP 访问。在生产环境不建议这样设置，一般要限定指定的 IP。

（3）将 **cluster.initial_master_nodes: ["node-1", "node-2"]** 的注释放开

修改完后保存然后重启 Elasticsearch

如果重启 Elasticsearch 报了以下这些错误：

```
ERROR: [4] bootstrap checks failed
[1]: max file descriptors [4096] for elasticsearch process is too low, increase to at least [65535]
[2]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
[3]: JVM is using the client VM [Java HotSpot(TM) Client VM] but should be using a server VM for the best performance
[4]: system call filters failed to install; check the logs and fix your configuration or disable system call filters at your own risk
```

你需要以下**4个**步骤来解决这些错误

[1]: 切换到root用户  编辑 /etc/security/limits.conf，在文件中添加如下配置

```
*                soft    nofile          65536
*                hard    nofile          131072
*                soft    nproc           2048
*                hard    nproc           409
```

如图所示：

![image](https://miansen.wang/assets/20190728175432.jpg)

[2]: 依然使用root用户 编辑 /etc/sysctl.conf，在最末尾添加如下配置

```
vm.max_map_count=2621441
```

保存后执行 sysctl -p /etc/sysctl.conf 使之生效

[3]: 编辑文件 JAVA_HOME\jre\lib\i386\jvm.cfg

将 `-server KNOWN` 与 `-client IF_SERVER_CLASS -server` 的位置对调

[4]: 编辑文件 config/elasticsearch.yml，将 `bootstrap.memory_lock: true` 的注释放开，把它的值改为 false，然后在下一行添加配置 `bootstrap.system_call_filter: false`

```
bootstrap.memory_lock: false 
bootstrap.system_call_filter: false
```

全部修改完后，需要退出当前用户，**重新登录**服务器才生效

## 基本概念

### 索引（Index）

这里的索引是一个名词，索引是 elasticsearch 存储数据的顶层单位，对应的是关系型数据库中的数据库，索引的名字必须是小写。

### 类型（Type）

类型对应的是关系型数据库中的表

> 在Elasticsearch 7.0.0或更高版本中创建的索引不再接受 _default_映射。在6.x中创建的索引将继续像以前一样在Elasticsearch 
> 6.x中运行。在7.0中的API中不推荐使用类型，对索引创建，放置映射，获取映射，放置模板，获取模板和获取字段映射API进行重大更改。

类型会在8.X版本彻底移除，不过下面的教程我还是保留了类型这个概念。

### 文档（Document）

Index 里面单条的记录称为 Document（文档），文档是一个可被索引的基础信息单元。文档以 JSON 格式来表示，对应的是关系型数据库中的行。

### 字段（Field）

文档的 key，对应的是关系型数据库中的列。

### 映射（Mapping）

映射是为每个字段匹配为一种确定的数据类型(string, number, booleans, date等)，可以了解为关系型数据库中的表结构。

我们新建索引的时候可以手动映射字段的类型，如果不手动映射，Elasticsearch 将自动为你映射。

不过一般情况我们还是手动映射字段的类型，因为有一些 string 类型的字段，我们不想让 Elasticsearch 做分词处理，比如说 "中国" 是一个整体，不能被拆分成 "中" 和 "国"，我们手动映射的时候可以指定 "not_analyzed" 属性，这样 Elasticsearch 就不会将这个字段分词了（后面会具体讲）。再比如 "2018-07-04" 是一个整体，表示某个特定的时间，也不能被分词，我们可以手动映射成 date 类型（date 类型的字段是不能被分词的），所以我们在新建索引时最好手动映射字段的类型。

Elasticsearch 与关系型数据库的对比：

|关系型数据库|Elasticsearch|
|:------------|:------------|
|数据库|索引|
|表|类型|
|行|文档|
|列|字段|
|映射|表结构|

### 分片（shard）

分片是 Elasticsearch 最小级别的"工作单元"，可以理解它是物理上真正存储数据的地方，而索引只是一个逻辑概念。

分片可以分为主分片（primary shard）和复制分片（replica shard），主分片是当前节点存储数据的地方，复制分片是其它节点备份数据的地方。

## 创建索引

文档的存储单元是索引，好比我们往关系型数据库里插入一条数据之前，得先把数据库创建出来。

由于 Elasticsearch 提供了 RESTful 风格的 API，所以我们可以发送 PUT 请求创建索引。

```shell
curl -X PUT 'http://localhost:9200/customer'
```

响应结果：

```json
{"acknowledged":true,"shards_acknowledged":true,"index":"customer"}
```

从响应结果中可以看到成功创建了索引，名字是 "customer"。

### 指定分片

创建索引可以指定分片

```shell
curl -X PUT 'http://localhost:9200/user' -H 'Content-Type:application/json' -d '{"settings":{"index":{"number_of_shards":5,"number_of_replicas":1}}}'
```

这条请求创建了名字为 "user" 的索引，并指定主分片为5个，复制分片为1个。

可以发起这样的请求获取索引的配置信息：

```shell
curl localhost:9200/user/_settings?pretty=true
```

pretty=true 的作用是返回美化后的 JSON 数据

响应结果：

```json
{
  "user" : {
    "settings" : {
      "index" : {
        "creation_date" : "1564528179814",
        "number_of_shards" : "5",
        "number_of_replicas" : "1",
        "uuid" : "uJedKPy9RRW9-lr6FyCE_A",
        "version" : {
          "created" : "7020099"
        },
        "provided_name" : "user"
      }
    }
  }
}
```

### 手动映射

创建索引时还可以手动映射字段的类型，好比我们在关系型数据库中创建表时指定表结构一样。

我们创建一个 "图书馆" 索引，该索引下有个 "图书" 类型，然后文档有图书名字（string）、作者（string）、价格（double）、出版日期（date）和简介（string）这5个字段。
图书名字和简介可以被分词，作者、价格和出版日期不能被分词。

我们可以这样映射：

```shell
curl -X PUT 'http://localhost:9200/library?pretty=true' -H 'Content-Type:application/json' -d '{"mappings":{"books":{"properties":{"bookName":{"type":"string","index":"analyzed"},"author":{"type":"string","index":"not_analyzed"},"price":{"type":"double","index":"not_analyzed"},"publishDate":{"type":"date","index":"not_analyzed"},"desc":{"type":"string","index": "analyzed"}}}}}'
```

请求体是这样的：

```json
{
	"mappings": {
		"books": {
			"properties": {
				"bookName": {
					"type": "string",
					"index": "analyzed"
				},
				"author": {
					"type": "string",
					"index": "not_analyzed"
				},
				"price": {
					"type": "double",
					"index": "not_analyzed"
				},
				"publishDate": {
					"type": "date",
					"index": "not_analyzed"
				},
				"desc": {
					"type": "string",
					"index": "analyzed"
				}
			}
		}
	}
}
```

可以发起这样的请求获取索引的映射信息：

```shell
curl localhost:9200/library/_mapping?pretty=true
```

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

## 查询文档

查询之前先索引一些文档，方便后面的查询操作

```shell
curl -X PUT 'http://localhost:9200/megacorp/employee/1' -H 'Content-Type:application/json' -d '{"first_name":"John","last_name":"Smith","age":25,"about":"I love to go rock climbing","interests":["sports","music"]}'
```

```shell
curl -X PUT 'http://localhost:9200/megacorp/employee/2' -H 'Content-Type:application/json' -d '{"first_name":"Jane","last_name":"Smith","age":32,"about":"I like to collect rock albums","interests":["music"]}'
```

```shell
curl -X PUT 'http://localhost:9200/megacorp/employee/3' -H 'Content-Type:application/json' -d '{"first_name":"Douglas","last_name":"Fir","age":35,"about":"I like to build cabinets","interests":["forestry"]}'
```

一共索引了3条文档

### 简单查询

发起 GET 请求，**索引/类型/id** 就可以定位到一个文档。

比如我们想要查询第一个员工的文档，用 SQL 是这么写的：

```sql
select * from employee a where a.id = 1;
```

在 elasticsearch 中，对应的请求是这样：

```shell
 curl -X GET 'http://localhost:9200/megacorp/employee/1?pretty=true'
```

url 后面加上 "pretty=true" 的作用是返回格式化后的 JSON，更容易阅读。

响应结果：

```json
{
  "_index" : "megacorp",
  "_type" : "employee",
  "_id" : "1",
  "_version" : 2,
  "_seq_no" : 1,
  "_primary_term" : 2,
  "found" : true,
  "_source" : {
    "first_name" : "John",
    "last_name" : "Smith",
    "age" : 25,
    "about" : "I love to go rock climbing",
    "interests" : [
      "sports",
      "music"
    ]
  }
}
```

原始的 JSON 文档包含在 "_source" 字段里，其他的字段是文档的**元信息**

### 简单搜索

我们想要搜索全部员工的文档，用 SQL 是这么写的：

```sql
select * from employee a;
```

在 elasticsearch 中，对应的请求是这样：

```shell
curl -X GET 'http://localhost:9200/megacorp/employee/_search?pretty=true'
```

还是用 megacorp 索引和 employee 类型查询，不过这次用 **_search** 取代原来的文档 ID

响应结果：

```json
{
  "took" : 14,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 3,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "1",
        "_score" : 1.0,
        "_source" : {
          "first_name" : "John",
          "last_name" : "Smith",
          "age" : 25,
          "about" : "I love to go rock climbing",
          "interests" : [
            "sports",
            "music"
          ]
        }
      },
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "2",
        "_score" : 1.0,
        "_source" : {
          "first_name" : "Jane",
          "last_name" : "Smith",
          "age" : 32,
          "about" : "I like to collect rock albums",
          "interests" : [
            "music"
          ]
        }
      },
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "3",
        "_score" : 1.0,
        "_source" : {
          "first_name" : "Douglas",
          "last_name" : "Fir",
          "age" : 35,
          "about" : "I like to build cabinets",
          "interests" : [
            "forestry"
          ]
        }
      }
    ]
  }
}
```

hits 数组中包含了我们所有的三个文档，默认情况下搜索会返回前10个结果。

### 查询字符串搜索

我们想要搜索姓氏中包含 **"Smith"** 的员工，用 SQL 是这么写的：

```sql
select * from employee a where a.last_name = 'Smith';
```

在 elasticsearch 中，可以使用查询字符串(query string)搜索。

在请求中依旧使用 _search 关键字，然后将查询语句传递给参数 **q=**，就像下面这条请求一样：

```shell
curl -X GET 'http://localhost:9200/megacorp/employee/_search?q=last_name:Smith&pretty=true'
```

响应结果：

```json
{
  "took" : 95,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 0.47000363,
    "hits" : [
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "1",
        "_score" : 0.47000363,
        "_source" : {
          "first_name" : "John",
          "last_name" : "Smith",
          "age" : 25,
          "about" : "I love to go rock climbing",
          "interests" : [
            "sports",
            "music"
          ]
        }
      },
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "2",
        "_score" : 0.47000363,
        "_source" : {
          "first_name" : "Jane",
          "last_name" : "Smith",
          "age" : 32,
          "about" : "I like to collect rock albums",
          "interests" : [
            "music"
          ]
        }
      }
    ]
  }
}
```

姓氏为 "Smith" 的员工就找到了，一共有两条。

### DSL语句查询

使用 DSL语句查询可以构建更加复杂、强大的查询。

DSL语句以 JSON 请求体的形式出现。

比如我们可以这样表示之前关于 "Smith" 的查询，用 DSL 语句查询可以这样写：

```shell
curl -X GET 'http://localhost:9200/megacorp/employee/_search?pretty=true' -H 'Content-Type:application/json' -d '{"query":{"match":{"last_name":"Smith"}}}'
```

响应的结果跟之前是一样的。只是这次我们不是使用参数的形式，而是使用**请求体**形式。

这个请求体的格式如下：

```json
{
    "query" : {
        "match" : {
            "last_name" : "Smith"
        }
    }
}
```

可以看到 match 字段里包含了我们想要搜索的信息。

#### 全文搜索

传入 **"match"** 字段可以实现全文搜索，类似于关系型数据库中的模糊查询。

使用 match 查询时，会先对输入进行分词，对分词后的结果进行查询，文档只要包含 match 查询条件的一部分就会被返回。

我们搜索所有喜欢 "rock climbing" 的员工，用 SQL 是这么写的：

```sql
select * from employee a 
where a.about like '%rock climbing"%'
or a.about like '%rock"%'
or a.about like '%climbing"%';
```

在 elasticsearch 中是这么写的：

```shell
curl -X GET 'http://localhost:9200/megacorp/employee/_search?pretty=true' -H 'Content-Type:application/json' -d '{"query":{"match":{"about":"rock climbing"}}}'
```

响应结果：

```json
{
  "took" : 95,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 1.4167402,
    "hits" : [
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "1",
        "_score" : 1.4167402,
        "_source" : {
          "first_name" : "John",
          "last_name" : "Smith",
          "age" : 25,
          "about" : "I love to go rock climbing",
          "interests" : [
            "sports",
            "music"
          ]
        }
      },
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "2",
        "_score" : 0.45895916,
        "_source" : {
          "first_name" : "Jane",
          "last_name" : "Smith",
          "age" : 32,
          "about" : "I like to collect rock albums",
          "interests" : [
            "music"
          ]
        }
      }
    ]
  }
}
```

> 默认情况下，Elasticsearch根据结果相关性评分来对结果集进行排序，所谓的「结果相关性评分」就是文档与查询条件的匹配程度。很显然，排名第一的John
> Smith的about字段明确的写到“rock climbing”。
> 但是为什么Jane Smith也会出现在结果里呢？原因是“rock”在她的abuot字段中被提及了。因为只有“rock”被提及而“climbing”没有，所以她的_score要低于John。

#### 短语搜索

有时候你想要确切的匹配若干个单词或者短语，想要查询同时包含"rock"和"climbing"（并且是相邻的）的员工记录。

要做到这个，我们用 SQL 是这么写的：

```sql
select * from employee a where a.about = 'rock climbing';
```

在 elasticsearch 中，只要将 match 查询变更为 match_phrase 查询即可：

```shell
curl -X GET 'http://localhost:9200/megacorp/employee/_search?pretty=true' -H 'Content-Type:application/json' -d '{"query":{"match_phrase":{"about":"rock climbing"}}}'
```

响应结果：

```json
{
  "took" : 75,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 1.4167402,
    "hits" : [
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "1",
        "_score" : 1.4167402,
        "_source" : {
          "first_name" : "John",
          "last_name" : "Smith",
          "age" : 25,
          "about" : "I love to go rock climbing",
          "interests" : [
            "sports",
            "music"
          ]
        }
      }
    ]
  }
}
```

#### 分页

在 SQL 中，构建分页我们是这样写的：

```sql
select * from employee a limit 0,10;
```

在 elasticsearch 中，我们可以传入 **"from"** 和 **"size"** 字段构建分页：

```shell
curl -X GET 'http://localhost:9200/megacorp/employee/_search?pretty=true' -H 'Content-Type:application/json' -d '{"query":{"match":{"last_name":"Smith"}},"from":1,"size":2}'
```

响应结果：

```json
{
  "took" : 18,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 0.47000363,
    "hits" : [
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "2",
        "_score" : 0.47000363,
        "_source" : {
          "first_name" : "Jane",
          "last_name" : "Smith",
          "age" : 32,
          "about" : "I like to collect rock albums",
          "interests" : [
            "music"
          ]
        }
      }
    ]
  }
}
```

from 参数(基于0)指定从哪个文档索引开始，size 参数指定从 from 参数开始返回多少文档。

注意，没有指定from，则默认值为0。

#### 排序

我们想搜索姓氏为 "Smith" 的文档，取第1到第10条，并按年龄降序排序，用 SQL 是这样写的：

```sql
select * from employee a 
where a.last_name = 'Smith'
order by a.age desc
limit 1,10;
```

在 elasticsearch 中，我们可以传入 **"sort"** 字段进行排序，比如下面这条请求：

```shell
curl -X GET 'http://localhost:9200/megacorp/employee/_search?pretty=true' -H 'Content-Type:application/json' -d '{"query":{"match":{"last_name":"Smith"}},"from":0,"size":10,"sort":{"age":{"order":"desc"}}}'
```

请求体的 json 是这样的：

```json
{
  "query": {
    "match": {
      "last_name": "Smith"
    }
  },
  "from": 0,
  "size": 10,
  "sort": {
    "age": {
      "order": "desc"
    }
  }
}
```

#### 返回指定字段

我们如果只想要查询特定字段，用 SQL 是这样写的：

```sql
select a.first_name,a.last_name from employee a where a.last_name = 'Smith';
```

在 elasticsearch 中，可以在请求体中传入 "_source" 字段：

```shell
curl -X GET 'http://localhost:9200/megacorp/employee/_search?pretty=true' -H 'Content-Type:application/json' -d '{"query":{"match":{"last_name":"Smith"}},"_source":["first_name","last_name"]}'
```

响应结果：

```json
{
  "took" : 23,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 0.47000363,
    "hits" : [
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "1",
        "_score" : 0.47000363,
        "_source" : {
          "last_name" : "Smith",
          "first_name" : "John"
        }
      },
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "2",
        "_score" : 0.47000363,
        "_source" : {
          "last_name" : "Smith",
          "first_name" : "Jane"
        }
      }
    ]
  }
}
```

#### bool 查询

bool 查询允许我们使用布尔逻辑，相当于 SQL 语句的 "and" 和 "or"

我们搜索名字等于 "Jane" **并且**姓氏等于 "Smith" 的员工

用 SQL 是这样写的：

```sql
select * from employee a where a.first_name = 'John' and a.last_name = 'Smith';
```

在 elasticsearch 中，是这样写的：

```shell
curl -X GET 'http://localhost:9200/megacorp/employee/_search?pretty=true' -H 'Content-Type:application/json' -d '{"query":{"bool":{"must":[{"match":{"first_name":"John"}},{"match":{"last_name":"Smith"}}]}}}'
```

请求体如下：

```json
{
  "query": {
    "bool": {
      "must": [{
        "match": {
          "first_name": "John"
        }
      }, {
        "match": {
          "last_name": "Smith"
        }
      }]
    }
  }
}
```

在这条请求中，**bool must** 子句指定了所有必须**都为true**的查询，也就是两个都为真，才会将文档视为匹配。

响应结果：

```json
{
  "took" : 26,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 1.4508328,
    "hits" : [
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "1",
        "_score" : 1.4508328,
        "_source" : {
          "first_name" : "John",
          "last_name" : "Smith",
          "age" : 25,
          "about" : "I love to go rock climbing",
          "interests" : [
            "sports",
            "music"
          ]
        }
      }
    ]
  }
}
```

搜索姓氏等于 "Fir" **或者** "Johnson" 的员工

用 SQL 是这样写的：

```sql
select * from employee a where a.last_name = 'Fir' or a.last_name = 'Johnson';
```

在 elasticsearch 中，是这样写的：

```shell
curl -X GET 'http://localhost:9200/megacorp/employee/_search?pretty=true' -H 'Content-Type:application/json' -d '{"query":{"bool":{"should":[{"match":{"last_name":"Fir"}},{"match":{"last_name":"Johnson"}}]}}}'
```

请求体如下：

```json
{
  "query": {
    "bool": {
      "should": [{
        "match": {
          "last_name": "Fir"
        }
      }, {
        "match": {
          "last_name": "Johnson"
        }
      }]
    }
  }
}
```

在这条请求中，**bool should** 子句指定了**只要有一个为true**的查询，也就是只要有一个为真，就会将文档视为匹配。

响应结果：

```json
{
  "took" : 5,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 0.9808292,
    "hits" : [
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "3",
        "_score" : 0.9808292,
        "_source" : {
          "first_name" : "Douglas",
          "last_name" : "Fir",
          "age" : 35,
          "about" : "I like to build cabinets",
          "interests" : [
            "forestry"
          ]
        }
      }
    ]
  }
}
```

搜索名字不等于 "John" 并且姓氏不等于 "Johnson" 的员工

用 SQL 是这样写的：

```sql
select * from employee a where a.first_name != 'John' and a.last_name != 'Johnson';
```

在 elasticsearch 中，是这样写的：

```shell
curl -X GET 'http://localhost:9200/megacorp/employee/_search?pretty=true' -H 'Content-Type:application/json' -d '{"query":{"bool":{"must_not":[{"match":{"first_name":"John"}},{"match":{"last_name":"Johnson"}}]}}}'
```

请求体如下：

```json
{
  "query": {
    "bool": {
      "must_not": [{
        "match": {
          "first_name": "John"
        }
      }, {
        "match": {
          "last_name": "Johnson"
        }
      }]
    }
  }
}
```

在这条请求中，**bool must_not** 子句指定了必须**都不为true**的查询，也就是两个都为假，才会将文档视为匹配。

响应结果：

```json
{
  "took" : 17,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 0.0,
    "hits" : [
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "2",
        "_score" : 0.0,
        "_source" : {
          "first_name" : "Jane",
          "last_name" : "Smith",
          "age" : 32,
          "about" : "I like to collect rock albums",
          "interests" : [
            "music"
          ]
        }
      },
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "3",
        "_score" : 0.0,
        "_source" : {
          "first_name" : "Douglas",
          "last_name" : "Fir",
          "age" : 35,
          "about" : "I like to build cabinets",
          "interests" : [
            "forestry"
          ]
        }
      }
    ]
  }
}
```

#### 过滤器查询

我们想要找到姓氏为 "Smith" 的员工，但是我们只想得到年龄大于30岁的员工。

用 SQL 是这样写的：

```sql
select * from employee a where a.last_name = 'Smith' and a.age > 30;
```

在 elasticsearch 中，可以使用过滤器（filter）查询，比如下面这条请求：

```shell
curl -X GET 'http://localhost:9200/megacorp/employee/_search?pretty=true' -H 'Content-Type:application/json' -d '{"query":{"bool":{"filter":{"range":{"age":{"gt":30}}},"must":{"match":{"last_name":"Smith"}}}}}'
```

请求体如下：

```json
{
  "query": {
    "bool": {
      "filter": {
        "range": {
          "age": {
            "gt": 30
          }
        }
      },
      "must": {
        "match": {
          "last_name": "Smith"
        }
      }
    }
  }
}
```

bool 查询包含一个 match 查询(查询部分)和一个 range 查询(筛选部分)

响应结果：

```json
{
  "took" : 95,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 0.47000363,
    "hits" : [
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "2",
        "_score" : 0.47000363,
        "_source" : {
          "first_name" : "Jane",
          "last_name" : "Smith",
          "age" : 32,
          "about" : "I like to collect rock albums",
          "interests" : [
            "music"
          ]
        }
      }
    ]
  }
}
```

#### 批量查询

我们想要批量查询 1、2、3 号员工，用 SQL 是这样写的：

```sql
select * from employee a where a.id in (1,2,3);
```

在 elasticsearch 中，可以用 **mget** API 查询：

```shell
curl -X GET 'http://localhost:9200/_mget?pretty=true' -H 'Content-Type:application/json' -d '{"docs":[{"_index":"megacorp","_type":"employee","_id":1},{"_index":"megacorp","_type":"employee","_id":2},{"_index":"megacorp","_type":"employee","_id":3}]}'
```

请求体是这样的：

```json
{
	"docs": [{
		"_index": "megacorp",
		"_type": "employee",
		"_id": 1
	}, {
		"_index": "megacorp",
		"_type": "employee",
		"_id": 2
	}, {
		"_index": "megacorp",
		"_type": "employee",
		"_id": 3
	}]
}
```

请求体是一个 docs 数组，数组的每个节点定义一个文档的 _index、_type、_id 元数据。

响应结果：

```json
{
  "docs" : [
    {
      "_index" : "megacorp",
      "_type" : "employee",
      "_id" : "1",
      "_version" : 2,
      "_seq_no" : 1,
      "_primary_term" : 2,
      "found" : true,
      "_source" : {
        "first_name" : "John",
        "last_name" : "Smith",
        "age" : 25,
        "about" : "I love to go rock climbing",
        "interests" : [
          "sports",
          "music"
        ]
      }
    },
    {
      "_index" : "megacorp",
      "_type" : "employee",
      "_id" : "2",
      "_version" : 1,
      "_seq_no" : 2,
      "_primary_term" : 2,
      "found" : true,
      "_source" : {
        "first_name" : "Jane",
        "last_name" : "Smith",
        "age" : 32,
        "about" : "I like to collect rock albums",
        "interests" : [
          "music"
        ]
      }
    },
    {
      "_index" : "megacorp",
      "_type" : "employee",
      "_id" : "3",
      "_version" : 1,
      "_seq_no" : 3,
      "_primary_term" : 2,
      "found" : true,
      "_source" : {
        "first_name" : "Douglas",
        "last_name" : "Fir",
        "age" : 35,
        "about" : "I like to build cabinets",
        "interests" : [
          "forestry"
        ]
      }
    }
  ]
}
```

响应体也包含一个docs数组，每个文档还包含一个响应，它们按照请求定义的顺序排列。