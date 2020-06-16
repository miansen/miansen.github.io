---
layout: post
title: 1.学习 Kafka - 搭建集群环境
date: 2020-05-18
categories: Kafka
tags: Kafka
author: 龙德
---

* content
{:toc}

## 前提条件

请准备 3 台虚拟机，以下是我准备的：

![image](https://miansen.wang/assets/20200519140014.png)

## 安装 JDK1.8

Zookeeper 和 Kafka 都依赖 JDK，所以安装 JDK 是必须的。

到官网 [https://www.oracle.com/java/technologies/javase-downloads.html](https://www.oracle.com/java/technologies/javase-downloads.html) 下载 JDK1.8。

下载完后，分别上传到每台虚拟机的 /usr/local/src 目录下。

我是用 Git 上传的。

```shell
scp jdk-8u251-linux-x64.tar.gz root@192.168.197.6:/usr/local/src
scp jdk-8u251-linux-x64.tar.gz root@192.168.197.10:/usr/local/src
scp jdk-8u251-linux-x64.tar.gz root@192.168.197.12:/usr/local/src
```

接着解压 jdk-8u251-linux-x64.tar.gz 压缩包到 /usr/local 目录。

```shell
tar zxvf /usr/local/src/jdk-8u251-linux-x64.tar.gz -C /usr/local/
```

然后配置每一台虚拟机的环境变量，编辑 profile 文件

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

到官网 [https://zookeeper.apache.org/releases.html](https://zookeeper.apache.org/releases.html) 下载 Zookeeper，我们下载 3.4.14 版本的。

![image](https://miansen.wang/assets/20200518165816.png)

![image](https://miansen.wang/assets/20200518165924.png)

下载完后，分别上传到每台虚拟机的 /usr/local/src 目录下。

我是用 Git 上传的。

```shell
scp zookeeper-3.4.14.tar.gz root@192.168.197.6:/usr/local/src
scp zookeeper-3.4.14.tar.gz root@192.168.197.10:/usr/local/src
scp zookeeper-3.4.14.tar.gz root@192.168.197.12:/usr/local/src
```

接着解压 zookeeper-3.4.14.tar.gz 压缩包到 /usr/local 目录。

```shell
tar zxvf /usr/local/src/zookeeper-3.4.14.tar.gz -C /usr/local/
```

然后配置每一台虚拟机的环境变量，编辑 profile 文件

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

添加日志文件夹路径

```shell
dataLogDir=/usr/local/zookeeper-3.4.14/logs
```

在文件末尾添加集群信息

```shell
server.1=192.168.197.6:2888:3888
server.2=192.168.197.10:2888:3888
server.3=192.168.197.12:2888:3888
```

![image](https://miansen.wang/assets/20200519113157.png)

创建数据文件夹

```shell
mkdir /usr/local/zookeeper-3.4.14/data
```

创建日志文件夹

```shell
mkdir /usr/local/zookeeper-3.4.14/logs
```

创建 myid 文件，在 myid 文件中添加本机的 server ID

```shell
echo "1" > /usr/local/zookeeper-3.4.14/data/myid
```

请注意，每台虚拟机的 myid 要跟 server ID 对应。

### 开放端口

每台虚拟机都开放以下端口，如果不开放的话，Zookeeper 集群无法与其它节点通信，启动会报错。

```shell
# 2181 端口对 client 端提供服务
firewall-cmd --zone=public --add-port=2181/tcp --permanent
# 2888 端口是集群内机器相互通信使用（由 Leader 负责监听此端口）
firewall-cmd --zone=public --add-port=2888/tcp --permanent
# 3888 端口是选举 leader 使用的
firewall-cmd --zone=public --add-port=3888/tcp --permanent
```
使端口生效

```shell
firewall-cmd --reload
```

### 启动 3 台虚拟机的 Zookeeper

使用以下命令分别启动每台虚拟机上的 Zookeeper。

```shell
/usr/local/zookeeper-3.4.14/bin/zkServer.sh start
```

### 查看 Zookeeper 的状态

```shell
/usr/local/zookeeper-3.4.14/bin/zkServer.sh status
```

如果输出以下信息，则说明 Zookeeper 集群搭建成功

192.168.197.6:

![image](https://miansen.wang/assets/20200519132502.png)

192.168.197.10:

![image](https://miansen.wang/assets/20200519132620.png)

192.168.197.12: 

![image](https://miansen.wang/assets/20200519132721.png)

通过以上图片可以知道，3 台虚拟机上的 Zookeeper 已经成功启起来了。其中 192.168.197.10 节点被选举为 leader 节点，其它两个节点是 follower 节点。每个节点都监听 3888 端口，当 leader 挂掉时，其它节点会通过此端口相互通信，选举出新的 leader。

2888 端口是集群内机器相互通信使用的，由 Leader 负责监听此端口。

## 搭建 Kafka 集群

### 安装 Kafka

到官网 [http://kafka.apache.org/downloads](http://kafka.apache.org/downloads) 下载 Kafka，我们下载 2.5.0 版本的。

![image](https://miansen.wang/assets/20200519134903.png)

![image](https://miansen.wang/assets/20200519134925.png)

下载完后，分别上传到每台虚拟机的 /usr/local/src 目录下。

我是用 Git 上传的。

```shell
scp kafka_2.13-2.5.0.tgz root@192.168.197.6:/usr/local/src
scp kafka_2.13-2.5.0.tgz root@192.168.197.10:/usr/local/src
scp kafka_2.13-2.5.0.tgz root@192.168.197.12:/usr/local/src
```

接着解压 kafka_2.13-2.5.0.tgz 压缩包到 /usr/local 目录。

```shell
tar zxvf kafka_2.13-2.5.0.tgz -C /usr/local/
```

然后配置每一台虚拟机的环境变量，编辑 profile 文件

```shell
vi /etc/profile
```

在最末尾添加以下内容

```shell
# set kafka environment
export KAFKA_HOME=/usr/local/kafka_2.13-2.5.0
export PATH=$KAFKA_HOME/bin:$PATH
```

保存 profile 文件，执行生效命令：

```shell
source /etc/profile
```

### 修改 Kafka 的配置文件

#### 修改 192.168.197.6 节点的配置文件

编辑配置文件

```shell
vi /usr/local/kafka_2.13-2.5.0/config/server.properties
```

修改配置如下（IP 地址应该根据实际情况填写）

```properties
# 当前节点在集群中的唯一标识，和 zookeeper 的 myid 性质是一样
broker.id=0
# 当前 kafka 对外提供服务的 ip 和端口，默认是 9092
listeners=PLAINTEXT://192.168.197.6:9092
# 消息存放的目录
log.dirs=/usr/local/kafka_2.13-2.5.0/logs
# zookeeper 集群的信息
zookeeper.connect=192.168.197.6:2181,192.168.197.10:2181,192.168.197.12:2181
# 连接 zookeeper 超时的最大时间
zookeeper.connection.timeout.ms=180000
```

#### 修改 192.168.197.10 节点的配置文件

编辑配置文件

```shell
vi /usr/local/kafka_2.13-2.5.0/config/server.properties
```

修改配置如下（IP 地址应该根据实际情况填写）

```properties
# 当前节点在集群中的唯一标识，和 zookeeper 的 myid 性质是一样
broker.id=1
# 当前 kafka 对外提供服务的 ip 和端口，默认是 9092
listeners=PLAINTEXT://192.168.197.10:9092
# 消息存放的目录
log.dirs=/usr/local/kafka_2.13-2.5.0/logs
# zookeeper 集群的信息
zookeeper.connect=192.168.197.6:2181,192.168.197.10:2181,192.168.197.12:2181
# 连接 zookeeper 超时的最大时间
zookeeper.connection.timeout.ms=180000
```

#### 修改 192.168.197.12 节点的配置文件

编辑配置文件

```shell
vi /usr/local/kafka_2.13-2.5.0/config/server.properties
```

修改配置如下（IP 地址应该根据实际情况填写）

```properties
# 当前节点在集群中的唯一标识，和 zookeeper 的 myid 性质是一样
broker.id=2
# 当前 kafka 对外提供服务的 ip 和端口，默认是 9092
listeners=PLAINTEXT://192.168.197.12:9092
# 消息存放的目录
log.dirs=/usr/local/kafka_2.13-2.5.0/logs
# zookeeper 集群的信息
zookeeper.connect=192.168.197.6:2181,192.168.197.10:2181,192.168.197.12:2181
# 连接 zookeeper 超时的最大时间
zookeeper.connection.timeout.ms=180000
```

然后每个节点都创建 logs 文件夹

```shell
mkdir /usr/local/kafka_2.13-2.5.0/logs
```

### 开放端口

每台虚拟机都开放以下端口

```shell
# 9092 端口是 Kafka 对 client 端提供服务的端口
firewall-cmd --zone=public --add-port=9092/tcp --permanent
```
使端口生效

```shell
firewall-cmd --reload
```

### 启动 3 台虚拟机的 Kafka

使用以下命令分别启动每台虚拟机上的 Kafka。

```shell
/usr/local/kafka_2.13-2.5.0/bin/kafka-server-start.sh /usr/local/kafka_2.13-2.5.0/config/server.properties &
```

上面那个命令是前台启动，会一直占用窗口，你也可以使用后台启动，命令如下：

```shell
/usr/local/kafka_2.13-2.5.0/bin/kafka-server-start.sh -daemon /usr/local/kafka_2.13-2.5.0/config/server.properties
```

如果输出以下信息，则说明启动成功

![image](https://miansen.wang/assets/20200519163500.png)

## 总结

最后来总结以下每台虚拟机安装的内容。

### 查看服务启动情况

使用这个命令查看服务启动情况：

```shell
jps
```

![image](https://miansen.wang/assets/20200519164832.png)

从上图可以看到安装了 Kafka 和 Zookeeper，并且都启动了。

### 查看端口占用情况

使用这个命令查看端口占用情况：

```shell
netstat -tunlp|egrep "(2181|2888|3888|9092)"
```

![image](https://miansen.wang/assets/20200519165440.png)


- 2888 端口：此端口是 Zookeeper 集群内机器相互通信使用的，由 Leader 节点负责监听此端口。
- 3888 端口：Zookeeper 集群的每个节点都监听 3888 端口，当 leader 挂掉时，其它节点会通过此端口相互通信，选举出新的 leader。
- 2181 端口：Zookeeper 对 client 端提供服务的端口。
- 9092 端口：Kafka 对 client 端提供服务的端口。

### 查看 Zookeeper 的目录情况

随便在一个节点上，使用以下命名，查看 Zookeeper 的目录情况。

使用客户端进入 Zookeeper

```shell
/usr/local/zookeeper-3.4.14/bin/zkCli.sh
```

![image](https://miansen.wang/assets/20200519180952.png)

使用 `ls /` 命令查看 Zookeeper 的目录情况。

![image](https://miansen.wang/assets/20200519181150.png)

上面的显示结果中：只有 zookeeper 目录是 Zookeeper 原生的，其它都是 Kafka 创建的。

关于各个目录的作用，后面再说。

至此，Kafka 的集群环境就搭建成功了。