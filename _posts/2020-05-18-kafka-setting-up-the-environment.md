---
layout: post
title: 1.学习Kafka-搭建环境
date: 2020-05-18
categories: Kafka
tags: Kafka
author: 龙德
---

* content
{:toc}

## 前提条件

请准备 3 台虚拟机，以下是我准备的：

|操作系统|ip|安装内容|
|-:|---:|-----------:|
|CentOS7-x86_64|192.168.197.6|jdk1.8、kafka、zookeeper|
|CentOS7-x86_64|192.168.197.10|jdk1.8、kafka、zookeeper|
|CentOS7-x86_64|192.168.197.12|jdk1.8、kafka、zookeeper|

## 安装 JDK1.8

使用以下命令为每台虚拟机下载 JDK1.8，我是下载到 /usr/local/src 目录。

```shell
wget --no-cookies --no-check-certificate --header "Cookie: gpw_e24=http%3A%2F%2Fwww.oracle.com%2F; oraclelicense=accept-securebackup-cookie" "http://download.oracle.com/otn-pub/java/jdk/8u141-b15/336fa29ff2bb4ef291e347e091f7f4a7/jdk-8u141-linux-x64.tar.gz"
```

接着解压下载的 jdk-8u251-linux-x64.tar.gz 压缩包到 /usr/local 目录。

```shell
tar zxvf /usr/local/src/jdk-8u251-linux-x64.tar.gz -C /usr/local/
```

然后配置环境变量，编辑 profile 文件

```shell
vi /etc/profile
```

在最末尾添加以下内容

```shell
# set jdk environmentexport
export JAVA_HOME=/usr/local/jdk1.8.0_251
export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$JAVA_HOME/bin:$PATH
```

保存 profile 文件，执行生效命令：

```shell
source /etc/profile
```

最后执行 java -version 命令，验证是否安装成功。输出以下内容则说明安装成功：

```shell
java version "1.8.0_251"
Java(TM) SE Runtime Environment (build 1.8.0_251-b08)
Java HotSpot(TM) 64-Bit Server VM (build 25.251-b08, mixed mode)
```

## 搭建 Zookeeper 集群

由于 Kafka 的 broker 的状态、消费者的消费状态、偏移量以及消费者组等是交给 Zookeeper 维护的，所以安装 Kafka 之前必须先安装 Zookeeper。

### 安装 Zookeeper

到官网 https://zookeeper.apache.org/releases.html 下载 Zookeeper，我们下载 3.4.14 版本的。

![image](https://miansen.wang/assets/20200518165816.png)

![image](https://miansen.wang/assets/20200518165924.png)

下载完后，分别上传到每台虚拟机的 /usr/local/src 目录下。

接着解压下载的 zookeeper-3.4.14.tar.gz 压缩包到 /usr/local 目录。

```shell
tar zxvf /usr/local/src/zookeeper-3.4.14.tar.gz -C /usr/local/
```

然后配置环境变量，编辑 profile 文件

```shell
vi /etc/profile
```

在最末尾添加以下内容

```shell
# set zookeeper environmentexport
export ZK_HOME=/usr/local/zookeeper-3.4.14
export PATH=$ZK_HOME/bin:$PATH
```

保存 profile 文件，执行生效命令：

```shell
source /etc/profile
```

### 修改 Zookeeper 的配置文件

将 zoo_sample.cfg 文件备份并重命名为 zoo.cfg

```shell
cp /usr/local/zookeeper-3.4.14/conf/zoo_sample.cfg /usr/local/zookeeper-3.4.14/conf/zoo.cfg
```

编辑 zoo.cfg 文件

```shell
vi zoo.cfg
```

修改数据文件夹路径

```shell
dataDir=/usr/local/zookeeper-3.4.14/data
```

在文件末尾添加以下内容

```shell
server.1=192.168.197.6:2888:3888
server.2=192.168.197.10:2888:3888
server.3=192.168.197.12:2888:3888
```

![image](https://miansen.wang/assets/20200518174229.png)

创建数据文件夹

```shell
mkdir /usr/local/zookeeper-3.4.14/data
```

接着创建 myid 文件，在 myid 文件中添加本机的 server ID

```shell
echo "1" > /usr/local/zookeeper-3.4.14/data/myid
```

请注意，每台虚拟机的 myid 要跟 server ID 对应。

### 启动 Zookeeper

使用以下命令分别启动每台虚拟机上的 Zookeeper。

```shell
/usr/local/zookeeper-3.4.14/bin/zkServer.sh start
```

### 查看 Zookeeper 的状态

```shell
/usr/local/zookeeper-3.4.14/bin/zkServer.sh status
```