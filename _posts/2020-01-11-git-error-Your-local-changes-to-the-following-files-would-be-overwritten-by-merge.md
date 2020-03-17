---
layout: post
title: Git报错：Your local changes to the following files would be overwritten by merge：xx
date: 2020-01-11
categories: 杂七杂八
tags: Git
author: 龙德
---

* content
{:toc}

用git pull来更新代码的时候，遇到了下面的问题：

```shell
$ git pull
Updating 725dd43..c34cb92
error: Your local changes to the following files would be overwritten by merge:
        src/main/java/cn/roothub/bbs/common/dao/jdbc/builder/DataSourceBuilder.java
Please commit your changes or stash them before you merge.
Aborting
```

出现这个问题的原因是其他人修改了 xxx.java 并提交到版本库中去了，而你本地也修改了xxx.java，这时候你进行 git pull 操作就好出现冲突了。

有两种解决方法：

**1、保留本地的修改**

```shell
git stash
git pull
git stash pop
```

通过 git stash 将工作区恢复到上次提交的内容，同时备份本地所做的修改，之后就可以正常 git pull 了，git pull 完成后，执行 git stash pop 将之前本地做的修改应用到当前工作区。

git stash: 备份当前的工作区的内容，从最近的一次提交中读取相关内容，让工作区保证和上次提交的内容一致。同时，将当前的工作区内容保存到Git栈中。

git stash pop: 从Git栈中读取最近一次保存的内容，恢复工作区的相关内容。由于可能存在多个 Stash 的 内容，所以用栈来管理，pop 会从最近的一个 stash 中读取内容并恢复。

git stash list: 显示 Git 栈内的所有备份，可以利用这个列表来决定从那个地方恢复。

git stash clear: 清空 Git 栈。此时使用 gitg 等图形化工具会发现，原来 stash 的哪些节点都消失了。

**2、放弃本地修改**

这种方法会丢弃本地修改的代码，而且不可找回。

```shell
git reset --hard
git pull
```

通过上述的方法 pull 之后，可能还会报这个错误：

```shell
error: The following untracked working tree files would be overwritten by merge:
        src/main/java/wang/miansen/roothub/common/dto/BaseDTO.java
Please move or remove them before you merge.
Aborting
```

这是由于一些 untracked working tree files（未跟踪的文件）引起的问题。

可以执行这个命令删除这些未跟踪的文件：

```shell
# 首先确认要删除的文件
git clean -fd -n
# 如果以上命令给出的文件列表是你想删除的， 那么接下来执行
git clean -d -fx
```

对于这个命令 `git clean -d -fx`，-f表示文件 -d表示目录, 如果还要删除.gitignore中的文件那么再加上-x。

**命名参考**

git删除未跟踪文件
 
- 删除 untracked files

`git clean -f`
 
- 连 untracked 的目录也一起删掉

`git clean -fd`
 
- 连 gitignore 的untrack 文件/目录也一起删掉 （慎用，一般这个是用来删掉编译出来的 .o之类的文件用的）

git clean -fdx
 
- 在用上述 git clean 前，墙裂建议加上 -n 参数来先看看会删掉哪些文件，防止重要文件被误删

```
git clean -nf
git clean -nfd
git clean -nfdx
```