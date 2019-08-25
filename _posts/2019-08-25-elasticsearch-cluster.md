---
layout: post
title: Elasticsearch-集群
date: 2019-08-25
categories: Elasticsearch
tags: Elasticsearch
author: 龙德
excerpt: 环境准备，请确保你已经有了以下环境，（1）3台虚拟机，我是用 VMware 搭建的虚拟机，每台虚拟机的地址如下：192.168.8.4、192.168.8.6、192.168.8.8（2）每台虚拟机都确保安装好了 Elasticsearch
---

* content
{:toc}

## 环境准备

请确保你已经有了以下环境

**（1）3台虚拟机**

我是用 VMware 搭建的虚拟机，每台虚拟机的地址如下：

192.168.8.4、192.168.8.6、192.168.8.8

**（2）每台虚拟机都确保安装好了 Elasticsearch**

## 集群搭建

搭建 Elasticsearch 集群前需要了解几个概念

**节点（Node）**

一个节点(node)就是一个 Elasticsearch 实例，就是说你在服务器上部署了一个 Elasticsearch 服务，那么这台服务器就可以称为节点。

**集群(cluster)**

由一个或多个节点组成，每个节点具有相同的 `cluster.name`

**主节点(master)**

集群中一个节点会被选举为主节点(master)，主节点不参与文档级别的变更或搜索，它只临时管理集群级别的一些变更，例如新建或删除索引、增加或移除节点等。

如果一个集群中只有一个节点，那么这个节点就会充当主节点的角色。

## 参考

[https://www.cnblogs.com/leeSmall/p/9220535.html](https://www.cnblogs.com/leeSmall/p/9220535.html)