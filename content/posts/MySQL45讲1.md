---
author: "Narcissus"
title: "MySQL基本介绍"
date: "2021-11-30"
description: "学习MySQL45讲散乱知识。"
tags: ["MySQL"]
categories: ["MySQL"]
---

## 基础架构：一条SQL查询语句是如何执行的？

MySQL的基本架构示意图如下：

![ScreenShot2021-11-20 11.16.31](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-11/ScreenShot2021-11-20%2011.16.31.png)

MySQL可以分为Server层和存储引擎层，不同存储引擎共用一个Server层。

- Server层包括连接器、查询缓存、分析器、优化器和执行器等。所有跨存储引擎的功能(比如：存储过程、触发器、视图等)都在这一层实现。
- 存储引擎层负责数据的存储和提取，架构模式是插件式的（支持InnoDB、Memory等）。建表的时候如果不指定存储引擎则默认使用InnoDB。

### 连接器

执行SQL语句时，首先需要连接MySQL服务端，**连接器负责跟客户端建立连接、获取权限、维持和管理连接**。使用`mysql -h ip -p port -u user -p`建立连接。这个过程中连接器会验证用户名和密码并查询出拥有的权限。

> 建立连接过程比较复杂，使用中尽量减少建立连接的动作，使用长连接。但是这样会导致内存占用太大，被系统强行杀掉（OOM），即显示MySQL异常重启。有两个解决方案：
>
> 1. 定期断开长连接。
> 2. 如果使用MySQL5.7或更新版本，可以在每次执行一个较大操作后，执行`mysql_reset_connection`来重新初始化连接资源。

### 查询缓存

建立连接后，就会执行输入的查询语句了。MySQL拿到一个查询请求后，会先到查询缓存中看看，之前是不是执行过这一条语句。**执行过的语句和结果可能会以key-value对形式缓存到内存中**,如果不在缓存中，就会继续后面的阶段，如果在缓存中则会直接返回结果。

> 查询缓存对于更新压力大的数据库来说，缓存命中率会非常低。因此在8.0版本直接删除了整个模块。低版本也可以“按需使用”，将参数`query_cache_type`设置为`DEMAND`，默认就不会使用查询缓存，需要可以显示指定。

### 分析器

没有命中查询缓存，MySQL需要对SQL语句做解析，知道你要做什么。分析器会先做“词法分析”，再做“语法分析”。

### 优化器

经过分析器后，再开始执行前，需要经过优化器处理。优化器是在表里面有多个索引的时候，决定使用哪一个索引；或者在一个语句关联多个表的时候，决定关联的顺序。该阶段完成后，执行方案就确定了。

### 执行器

开始执行前，会先判断你对这个表有没有执行查询的权限，如果没有会返回没有权限的错误；如果有则会根据表的引擎定义，使用指定的引擎提供的接口查询结果并返回。

## 日志系统：一条SQL更新语句是如何执行的？

查询语句的流程更新语句也会走一遍，最后通过执行器更新。（**注：在一个表上更新的时候，跟这个表有关的查询缓存会失效，所以会把相关表的所有缓存结果都清空**）。与查询流程不一样的是，更新流程还涉及两个重要的日志模块：**重做日志(`redo log`)和归档日志(`binlog`)**。

### redo log

**redo log是InnoDB引擎特有的日志**，redo log用到了MySQL里常说的WAL(Write-Ahead Logging)技术，关键点就是先写日志，再写磁盘。具体来说就是，当有一条记录需要更新的时候，InnoDB引擎会先把记录写到redo log 中，并更行内存，此时就算更新完成。同时InnoDB会在适当的时候将这个操作记录更新到磁盘里。

InnoDB的redo log大小是固定的，从头开始写，写到末尾就又回到开头循环写，如下图：

![](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-11/ScreenShot2021-11-20%2015.25.38.png)

`write pos`是当前记录的位置，`checkpoint`是当前要擦除的位置，擦除记录前要把记录更新到数据文件。

> 注意：有了redo log，InnoDB就可以保证即使数据库发生异常重启，之前提交的数据也不会丢失。这个能力称为**crash-safe**。

### binlog

Server层也有自己的日志，称为binlog（归档日志）。

> 两种日志有以下不同点：
>
> 1. redo log是InnoDB引擎特有的；binlog是MySQL的Server层实现的，所有引擎都可以使用。
> 2. redo log是物理日志，记录的是“在某个数据页上做了什么修改”；binlog是逻辑日志，记录语句的原始逻辑。
> 3. redo log是循环写，空间固定会用完；binlog可以追加写，不会覆盖以前的日志。

执行器和InnoDB在执行更新语句时的内部流程，如下图（浅色框表示在InnoDB内部执行，深色框表示在执行器中执行）：

![ScreenShot2021-11-20 15.38.44](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-11/ScreenShot2021-11-20%2015.38.44.png)

### 两阶段提交

redo log的写入拆成了两个步骤：prepare和commit，这就是两阶段提交。redo log和binlog都可以用于表示事务的提交状态，而**两阶段提交就是让这两个状态保持逻辑上的一致**。

