---
title: Let's Build a Simple Database - How Does a Database Work
date: 2021-03-27 17:12:13
tags:
- 翻译
- Database
categories:
- 翻译
---

> 原文地址：https://cstack.github.io/db_tutorial/
> 原文作者： [cstack](https://github.com/cstack)
> 译者：[Win-Man](https://github.com/Win-Man)

## 一个数据库是如何工作的？

* 在内存中、在磁盘上数据存储格式是怎么样的？
* 什么时候数据库从内存转移到磁盘？
* 为什么每个表只能有一个主键？
* 事务回滚是如何工作的？
* 索引的格式是怎么样的？
* 什么时候会发生全表扫以及全表扫是如何工作的？
* 预处理语句是以什么形式存储的？

简而言之，问题就是数据库是怎么工作的？

为了搞明白这个问题，我用 C 写了一个 [sqlite](https://www.sqlite.org/arch.html) 的克隆版本，并且记录了我的整个过程。

## 目录

* [Part1 - 介绍并设置 REPL](https://cstack.github.io/db_tutorial/parts/part1.html)
* [Part2 - 世界上最简单的 SQL 编译器以及虚拟机](https://cstack.github.io/db_tutorial/parts/part2.html)
* [Part3 - 一个只允许追加、单表、内存数据库](https://cstack.github.io/db_tutorial/parts/part3.html)
* [Part4 - 我们的第一个测试(Bug)](https://cstack.github.io/db_tutorial/parts/part4.html)
* [Part5 - 磁盘持久化](https://cstack.github.io/db_tutorial/parts/part5.html)
* [Part6 - 抽象游标](https://cstack.github.io/db_tutorial/parts/part6.html)
* [Part7 - B 树介绍](https://cstack.github.io/db_tutorial/parts/part7.html)
* [Part8 - B 树叶子节点结构](https://cstack.github.io/db_tutorial/parts/part8.html)
* [Part9 - 二进制搜索以及重复键](https://cstack.github.io/db_tutorial/parts/part9.html)
* [Part10 - 叶子节点的分裂](https://cstack.github.io/db_tutorial/parts/part10.html)
* [Part11 - B 树的递归搜索](https://cstack.github.io/db_tutorial/parts/part11.html)
* [Part12 - 扫描 Multi-Level B 树](https://cstack.github.io/db_tutorial/parts/part12.html)
* [Part13 - 分裂之后更新父节点](https://cstack.github.io/db_tutorial/parts/part13.html)

"What I cannot create, I do not understand" - Richard Feynman

![sqlite architecture](https://cstack.github.io/db_tutorial/assets/images/arch2.gif)

<center>sqlite architecture (https://www.sqlite.org/arch.html)</center>