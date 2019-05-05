---
title: MySQL数据恢复新姿势
date: 2018-11-30 21:12:13
tags:
- MySQL
categories:
- MySQL
---

## 一、背景

### 1.1、需求来源
> 需要给客户将生产环境的数据恢复到测试环境，但是线上环境的xtrabackup全备数据量太大，拷贝下来比较麻烦，并且需要恢复的数据只有整库中的两张表的数据，所以客户只是将全备中的对应表的ibd文件以及frm文件拷贝下来了，要求根据ibd文件以及frm文件进行数据恢复。

### 1.2、环境介绍

* 数据库版本：MySQL-5.7.22
* 数据库要求：innodb_file_per_table=1

## 二、解决方案步骤

### 2.1、准备工作

* 准备好需要进行数据恢复的表ibd文件以及frm文件
* 安装一个新的MySQL实例
 * innodb_file_per_table
 * 
安装步骤省略

### 2.2、表结构恢复

* 在数据库中创建一张表名与被恢复表表名一致的表，表结构不限制

![](https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20190505160614.png)

* 将新建表的ibd文件以及frm文件拷贝到tmp目录下备份

```
# cp message_index.* /tmp/
```

* 使用被恢复的frm文件替换新创建的同名表的frm文件

```
# cp /data2/message_index.frm ./
cp：是否覆盖"./message_index.frm"？ y
# chown -R mysql:mysql ./*
```

* 在数据库中执行show create table语句
注意需要在show create table 查看表结构之前执行flush tables语句，因为如果message_index表之前被打开过，那么表结构会被缓存在内存中，show create table不会报错，也就无法从错误日志中拿到我们需要的信息。

![](https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20190505160723.png)

* 查看error.log，获取被恢复表的字段数
错误日志中会打印我们需要恢复的表的字段数，这边可以看到我们需要恢复的表中含有6个字段

![](https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20190505160758.png)

* 删除message_index表，并重新创建message_index表

从上面的步骤中我们知道被恢复表中含有6个字段，所以重新创建的message_index表需要含有6个字段，字段名以及字段类型不限制

![](https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20190505160924.png)

* 再次使用被恢复的frm文件替换新创建的同名表的frm文件

```
# cp /data2/message_index.frm ./
cp：是否覆盖"./message_index.frm"？ y
# chown -R mysql:mysql ./*
```

* 在MySQL配置文件中添加innodb_force_recovery=6，并重启数据库

![](https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20190505161041.png)

* 通过show create table语句拿到message_index表的表结构

![](https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20190505161054.png)

至此，我们就拿到了我们需要进行恢复的表的表结构。
### 2.3、表数据恢复

拿到表结构之后，表数据恢复步骤相对表结构恢复步骤而言会简单一些

* 将innodb_force_recovery=6从配置文件中去掉、使用/tmp目录下的ibd文件以及frm文件覆盖当前的对应文件、重启数据库
* 在数据库中按照获取到的表结构新建一张message_index表
* 执行`alter table discard tablespace`语句
* 
```
mysql> alter table message_index discard tablespace;
```

![](https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20190505161116.png)

* 将要恢复的表的ibd文件拷贝到当前库下，并更改属主以及属组
```
# chown -R mysql:mysql ./*
```
* 执行alter table import tablespace语句

```
mysql> alter table message_index import tablespace;
```

![](https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20190505161139.png)

可以看到数据已经都恢复回来了。


## 三、总结

* 以上我们通过xtrabackup全备中的ibd文件以及frm文件恢复了数据，这样也就代表着xtrabackup就算备份失败，只要有部分ibd文件以及frm文件保证完好，MySQL也是可以进行数据恢复的，在极端情况下也能尽可能的减少损失。但是由于xtrabackup是通过记录redo日志的方式来保存备份过程中产生的增量数据，这一部分增量数据目前还没有办法恢复。
* 导入表空间的方式，也给MySQL大数据量迁移方案提供了一种思路：直接拷贝ibd文件的方式






> 博客地址：https://win-man.github.io/  
> 公众号：欢迎关注  

![](https://user-gold-cdn.xitu.io/2018/8/16/165435ce71d2b88b?w=258&h=258&f=jpeg&s=26568)
