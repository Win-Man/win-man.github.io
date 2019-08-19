---
title: MySQL日常问题两则
date: 2019-08-12 11:08:13
tags:
- MySQL
- emoji
- DDL
categories:
- MySQL
---

## 1. 前言
近期在给开发小伙伴解决问题的时候，收集了两个个人觉得比较有意思的问题给大家分享一下。一个是在执行ALTER TABLE ADD COLUMN语句时，报了Duplicate entry的错误；另一个是关于在MySQL中正确存取emoji表情的问题。

## 2. ALTER TABLE ADD COLUMN报Duplicate entry错误
### 2.1 问题描述
某日系统上线，接到开发小伙伴电话说在上线时，执行一个增加字段的DDL语句脚本时，报错了，错误如下:
`ERROR 1062 (23000) at line 1: Duplicate entry 'UR000021426347' for key 'T_CAP_CUST_MIDDLE_INFO_UNIQ_INDEX'`
根据错误提示的条件去数据库中查询却只能查到一条记录，并没有重复记录。DDL脚本无法执行，影响后续上线步骤了。当时由于不在现场，了解到的信息只有：

1. DDL语句脚本中只有两条DDL语句，且都是添加字段的语句 
2. 脚本正常运行只需要40-70秒，
3. 当时并不是停业窗口。

考虑到数据库版本为5.6.34，添加字段并不会阻塞DML操作，让开发小伙伴再运行一次试试，结果这次执行成功了，并没有报冲突的错误。线上问题顺利解决，具体原因得线下分析了。虽说解决问题是主要矛盾，但是搞清楚问题原因有着更深层次的意义。

### 2.2 原因定位
下面就是到了寻找问题原因的时候了，为什么同样的DDL语句脚本第一次执行的时候报了Duplicate entry错误，第二次却顺利运行了。其实问题原因很好找，打开Google，输入关键字`mysql alter table add column duplicate entry`，搜索结果中很多关键字完全匹配的链接，说明很多人遇到过相同问题。搜索结果中一眼就看到一个链接[MySQL Bugs:#76895:Adding new column OR Drop column causes duplicate PK error](https://bugs.mysql.com/bug.php?id=76895)，看到MySQL Bug就莫名兴奋。通过该Bug链接了解到该问题是Online DDL的一个限制问题，官方认为该问题是一种限制，并不是Bug，所以目前为止还没有得到解决。

> When running an online DDL operation, the thread that runs the ALTER TABLE statement applies an online log of DML operations that were run concurrently on the same table from other connection threads. When the DML operations are applied, it is possible to encounter a duplicate key entry error (ERROR 1062 (23000): Duplicate entry), even if the duplicate entry is only temporary and would be reverted by a later entry in the online log. This is similar to the idea of a foreign key constraint check in InnoDB in which constraints must hold during a transaction.

解释一下就是当执行Oline DDL操作时，MySQL实际上是将DML缓存（该缓存大小由变量 innodb_online_alter_log_max_size控制，默认128M）起来，等DDL执行完成后再将缓存中的DML重新应用到表上。如果有别的线程执行了DML操作，在DDL完成后，应用DML时，可能会出现duplicate entry错误。

### 2.3 实验验证
上面通过Google找到了理论上可能能解释问题的原因描述，但是还没有实际验证，所以接下来就是线下复现环节。先去找开发同事问了下线上报错的表只有一种操作`insert into ... on duplicate key...`，且报Duplicate entry的字段上有唯一索引。如果没有冲突的记录则插入，否则就更新。那么验证测试步骤也比较简单了，找一张测试表，执行ALTER TABLE ADD COLUMN操作，并同时执行insert into...on duplicate key...操作，观察DDL语句是否会有报错。

#### 2.3.1 实验环境
* redhat-6.7/redhat-7.4
* MySQL-5.6.34/MySQL-5.7.22

#### 2.3.2 操作步骤

* 准备测试环境

<img src="https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20190809095927.png" height="50%" width="50%" />

<img src="https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20190809095941.png" height="50%" width="50%" />

col1字段的值与id字段的值是一致的，test表中共有1600W+条记录。

* 执行DDL同时执行insert into ... on duplicate key...操作

| 序号 | 会话1 | 会话2 |
| ----- | ------ | ------- |
| 1 | select * from test limit 10;<br><img src="https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20190809095447.png" height="600px" width="200px" />| - |
| 2 | - | alter table test add column address varchar(50) default 'HangZhou'; |
| 3 | insert into test (name,col1) values('san.zhang',7) on duplicate key update col1=29;<br>ERROR 1062 (23000): Duplicate entry '29' for key 'col1' | - |
| 4 | - | DDL执行报错，检查发现DDL执行失败<br>ERROR 1062 (23000): Duplicate entry '7' for key 'col1' |
| 5 | 查询test表中数据<br><img src="https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20190809100059.png" height="400px" width="600px" /> | - |
顺利的复现了线上的问题现象，那说明当时线上就是因为DML更新了相同的唯一属性字段键值导致DDL执行失败，报错。测试过程中想到insert into... on duplicate key...不行，那么replace into 会不会也一样导致问题呢，于是就同样对replace into语句进行了测试。

* 执行DDL同时执行replace into操作

| 序号 | 会话1 | 会话2 |
| ----- | ------- | ------ |
| 1 | select * from test limit 10;<br><img src="https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20190809104744.png" height="600px" width="200px" /> | - |
| 2 | - | alter table test add column address varchar(50) default 'HangZhou'; |
| 3 | replace into test(id,name,col1) values(1,'gang.shen',13);<br>Query OK, 3 rows affected (0.00 sec) | - |
| 4 | - | DDL执行报错，检查发现DDL执行失败<br>ERROR 1062 (23000): Duplicate entry '13' for key 'col1' |
| 5 | select *  from test where col1=13;<br><img src="https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20190809104942.png" height="100px" width="600px" /> | - |

* 后续以同样方式测试了UPDATE、INSERT操作，实验证明都会影响DDL操作的正确执行，在5.7.22版本上也是一样的现象。这边出于篇幅考虑就不将测试过程给出了，感兴趣的读者可以自己实验一把，并且可以验证下在8.0版本中是否仍然存在该问题。


 
### 2.4 小结

在这对这个问题做几点总结：

* 问题原因：Oline DDL的原理简单一点理解就是将DML操作缓存起来，等到DDL执行完成后重新应用缓存中的DML语句，如果在Oline DDL执行过程中，DML操作产生了Duplicate entry错误，并不会直接影响DDL操作，而是在DDL执行完成最终应用DML时报错，导致DDL执行失败。关于Oline DDL执行步骤可以参考：[https://yq.aliyun.com/articles/282290](https://yq.aliyun.com/articles/282290)。
* 在MySQL Bug网站上，官方人员回复该现象并不是Bug，而是一种限制。但是个人认为是可以做一些改善的，因为在测试insert into ... on duplicate key...以及update和insert语句时，对于执行DML操作的客户端已经直接返回报错了，但是从现象上看MySQL仍然将报错的DML语句放到了Oline DDL的缓存中，如果直接将报错语句从缓存中去除则不会影响DDL的正常执行。这只是个人简单认为，深入的话需要可以通过代码去确认。
* Google是位好老师



## 3. MySQL存取emoji的正确姿势
### 3.1 问题描述
开发小伙伴在测试环境测试过程中，需要往MySQL数据库中插入emoji表情，但是发现一套测试环境可以插入成功，另一套测试环境不行，且插入成功的那套环境数据库将数据查询出来是`????`乱码。经过与开发确认后了解到以下几点信息：

* 表中只有一个字段需要存放emoji表情，开发对该字段单独设置了utf8mb4字符集
* 可以成功插入成功的测试环境数据库版本为5.6.34，无法插入的测试环境数据库版本为5.7.22
* 应用连接代码中使用的是utf8字符集

那其实需要解决的问题其实是两个问题：1、5.6版本下emoji存取乱码问题 2、5.7版本emoji无法插入的问题

### 3.2 实验验证
#### 3.2.1 MySQL-5.6 emoji存取乱码问题
我们先来看5.6版本下emoji存储乱码的问题，理论上从数据库角度考虑，字段字符集已经设置为utf8mb4，应用使用的是utf8字符集连接，插入emoji需要utf8mb4字符集，多半是连接字符集设置的问题。大胆假设已经完成，接下来就是小心求证的过程。

> 实验环境
> * 数据库版本：5.6.44
> * sql_mode='NO_ENGINE_SUBSTITUTION'
> * default-character-set=utf8

* 使用utf8字符集连接MySQL，并检查参数设置

<img src="https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20190809212132.png" height="50%" width="50%" />

* 创建测试表

<img src="https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20190809212449.png" height="50%" width="50%" />

* 插入emoji表情

<img src="https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20190809212941.png" height="50%" width="50%" />

表情的确能插入数据库，但是insert时有两条warning记录提示是invalid的字符串，并且select查询出来的数据也是一样`????`乱码

* 使用utf8mb4字符集插入emoji表情并查询

<img src="https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20190809213245.png" height="50%" width="50%" />
可以看到连接使用utf8mb4字符集插入、查询emoji表情都是正常的，插入时也没有warning提示，那说明emoji乱码的问题就是因为连接字符集设置不合理导致的。



#### 3.2.2 MySQL-5.7 emoji无法插入的问题

定位了乱码的问题，再来看看emoji无法插入的问题。

> 实验环境
> * 数据库版本：5.7.22
> * sql_mode='ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION'
> * default-character-set=utf8

* 使用utf8字符集连接MySQL，并检查参数设置

<img src="https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20190809214109.png" height="50%" width="50%" />

* 创建测试表

<img src="https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20190809212449.png" height="50%" width="50%" />

* 插入emoji表情

<img src="https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20190809214254.png" height="50%" width="50%" />

插入的时候直接报错了，同样的字符集，同样的语句，5.6版本下emoji就能插入，5.7就直接报错了。5.6->5.7数据库版本问题，如果踩坑比较多的话，还是比较容易联想到是不是sql_mode参数的问题，因为5.7中sql_mode参数默认值多了很多项，对语句的限制加强了很多。

* 查看sql_mode参数默认值

```
mysql> show variables like '%sql_mode%'\G
*************************** 1. row ***************************
Variable_name: sql_mode
        Value: ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
1 row in set (0.00 sec)
```

查看官方文档关于各个sql_mode选项的值，找到了怀疑对象STRICT_TRANS_TABLES，表示开启严格模式，严格模式下如果插入的数据不在范围之类会报错中断语句。

* 修改sql_mode参数值，再次插入emoji表情测试

```
mysql> set session sql_mode='ONLY_FULL_GROUP_BY,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION';
Query OK, 0 rows affected, 1 warning (0.00 sec)
```

<img src="https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20190809220506.png" height="50%" width="50%" />

可以看到修改sql_mode值后，现象与5.6版本下一致

* 使用utf8mb4字符集插入emoji表情并查询

<img src="https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20190809213245.png" height="50%" width="50%" />

### 3.3 小结
在此对emoji问题做一下小结：

* MySQL中存储emoji不仅需要将表结构中字段字符集设置为utf8mb4，还需要考虑连接字符集的问题
* 建议数据库使用5.7.22及以上版本，并且sql_mode开启严格模式，这样当数据出现异常可以及时发现
* 在sql_mode开启严格模式的情况下，应用端连接数据库也需要调整为utf8mb4字符集才可以正常插入emoji表情
* 建议将默认字符集设置为utf8mb4，做到库、表、字段字符集都统一，utf8mb4是utf8的超集，能兼容更多的字符，而且在MySQL-8.0中已经将utf8mb4设置为了默认的字符集。

> 博客地址：https://win-man.github.io/  
> 公众号：欢迎关注  

![](https://user-gold-cdn.xitu.io/2018/8/16/165435ce71d2b88b?w=258&h=258&f=jpeg&s=26568)


