---
author: "Narcissus"
title: "MySQL:索引"
date: "2021-11-30"
description: "学习MySQL45讲索引相关知识。"
tags: ["MySQL"]
categories: ["MySQL"]
---

## 深入浅出索引

索引的出现就是为了提高数据查询的效率，就像书的目录一样。

### 索引常见模型优劣分析

- 哈希表

哈希表是以一种键值对存储的数据结构，**适用于只有等值查询的场景**。

- 有序数组

**有序数组在等值查询和范围查询场景中性能都非常优秀，但只适用于静态存储索引**。因为修改需要挪动记录，成本太高。

- 搜索树

二叉树是搜索树效率最高的，但实际大多数数据库存储并不使用二叉树。原因是，**索引不止存在内存中，还要写到磁盘上**。

> 受限与磁盘的读写性能，减少单次查询尽量少的读磁盘，就必须让查询过程访问尽量少的数据块，因此不应该使用二叉树，而要使用N叉树。

**N叉树由于读写上的性能优点，以及适配磁盘的访问模式，已经被广泛应用在数据库引擎中**。

### InnoDB的索引模型

在InnoDB中，表都是根据主键顺序以索引的形式存放的，这种存储方式的表称为**索引组织表**。**InnoDB使用B+树索引模型**，每一个索引在InnoDB里面对应一棵B+树。

有一个主键列为ID的表，表中有字段k,并且在k上有索引，则两棵树如下：

![](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-11/ScreenShot2021-11-20%2019.49.31.png)

可以看出，根据叶子节点的内容，**索引类型分为主键索引和非主键索引**。

- 主键索引的叶子节点存储整行数据，在InnoDB中，主键索引也被称为聚簇索引。
- 非主键索引的叶子节点内容是主键的值，在InnoDB中，非主键索引也被称为二级索引。

> 基于主键索引和普通索引的查询有什么区别？
>
> **基于非主键索引的查询需要多扫描一棵索引树**，即先通过非主键索引查询到对应主键，再通过主键查询到对应的记录。

### 索引维护

B+树为了维护索引的有序性，在插入新值的时候需要做必要的维护。在这个过程中可能会需要一个新的数据页，挪动部分数据过去，这个过程称为**页分裂。**

> 页分裂会影响性能和数据页的利用率，当利用率很低之后，会将数据页合并。

自增主键的优点：

1. **其插入数据的模式，符合递增插入的场景，不涉及挪动其他记录，不会触发页分裂，写数据成本较低**。
2. **主键长度越小，普通索引的叶子节点就越小，普通索引占用空间就越小**。

### 回表

一条查询语句如果需要**回到主索引树搜索的过程，称为回表**，如果需要查询结果所需要的数据只有在主键索引上有，就不得不回表。下面介绍一些索引优化，避免回表过程。

### 覆盖索引

在查询的结果里，索引已经“覆盖了”我们的查询需求，则称为**覆盖索引**。由于**覆盖索引可以减少树的搜索次数，显著提升查询性能**，所以使用覆盖索引是一个常见的性能优化手段。

### 最左前缀原则

B+树这种索引结构，可以利用索引的最左前缀，来定位记录。例如有联合索引(name, age)如下：

![ScreenShot2021-11-30 18.55.15](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-11/ScreenShot2021-11-30%2018.55.15.png)

不只是索引的全部定义，只要满足最左前缀，就可以利用索引来加速检索，**原则就是，如果通过调整顺序，可以少维护一个索引，那么这个顺序往往就是需要优先考虑采用的**。

### 索引下推

现在有一个需求：检索出表中“名字第一个字是张，而且年龄是10岁的所有男孩”，如果没有索引下推，则流程如下：

![ScreenShot2021-11-30 19.01.49](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-11/ScreenShot2021-11-30%2019.01.49.png)

在MySQL5.6引入了索引下推优化，**可以在索引遍历过程中，对索引中包含的字段先做判断，直接过滤掉不满足条件的记录，减少回表次数**。流程图如下：

![ScreenShot2021-11-30 19.03.41](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-11/ScreenShot2021-11-30%2019.03.41.png)

这样，对于不等于10的记录，直接判断并跳过，只需要对ID4、ID5这两条记录回表取数据判断，这样就只需要回表2次。