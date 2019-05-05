---
title: MySQL5.7中的sql_mode默认值
date: 2018-10-30 17:12:13
tags:
- MySQL
categories:
- MySQL
---
[toc]

## 简介
在正常项目开发过程中，如果MySQL版本从5.6升级到5.7版本。作为DBA在考虑数据库版本升级带来的影响时，一般会有几个注意点：
1. `sql_mode` 默认值的改变
2. `optimizer_switch` 值的改变
3. 备库升级影响主备复制
本文主要内容是MySQL升级到5.7版本之后，由于默认的 `sql_mode` 值带来的坑以及对应的解决方案。
## 案例一：ONLY_FULL_GROUP_BY
### 问题描述
MySQL版本从5.6升级至5.7之后，部分SQL执行报错，报错信息如下：
```sql
ERROR 1055 (42000): Expression #3 of XXXXXX list is not in GROUP BY clause and contains nonaggregated column ‘XXXXX.XXXXXX’ which is not functionally dependent on columns in GROUP BY clause; this is incompatible with sql_mode=only_full_group_by
```
这个问题原因在于从5.6升级至5.7版本后 `sql_mode` 默认值发生了改变，在5.7版本的 `sql_mode` 默认值中有意向 `ONLY_FULL_GROUP_BY`，该选项的含义表示：对于使用 `GROUP BY` 进行查询的SQL，不允许 `SELECT` 部分出现 `GROUP BY` 中未出现的字段，也就是 `SELECT` 查询的字段必须是 `GROUP BY` 中出现的或者使用聚合函数的或者是具有唯一属性的。
### 解决方案
* 方案一（不推荐）：修改5.7版本 `sql_mode` 值，将 `ONLY_FULL_GROUP_BY` 去掉
 `ONLY_FULL_GROUP_BY` 是加强SQL规范的，其目的是让SQL查询出来的结果更符合规范，更准确。
如果没有 `ONLY_FULL_GROUP_BY` 规范限制，那么则能允许以下SQL的执行：`SELECT a,b,c FROM t GROUP BY a`。SQL按照a字段值进行分组，当同一个a字段值对应多个b或者c值时，查询结果中的b,c值是不确定的。
* 方案二：对于不符合`ONLY_FULL_GROUP_BY`限制的字段，添加unique索引
* 方案三：改写SQL，按照规范编写SQL
* 方案四：使用ANY_VALUE()，对于不符合`ONLY_FULL_GROUP_BY`的字段使用ANY_VALUE()函数，让MySQL跳过`ONLY_FULL_GROUP_BY`检测
```sql
mysql> SELECT name, address, MAX(age) FROM t GROUP BY name;
ERROR 1055 (42000): Expression #2 of SELECT list is not in GROUP
BY clause and contains nonaggregated column 'mydb.t.address' which
is not functionally dependent on columns in GROUP BY clause; this
is incompatible with sql_mode=only_full_group_by
mysql>SELECT name, ANY_VALUE(address), MAX(age) FROM t GROUP BY name;
```
## 案例二：NO_ZERO_DATE & NO_ZERO_IN_DATE & time_zone
### 问题描述
#### 排错阶段一
MySQL版本从5.6升级至5.7之后，创建表的过程中失败：
```sql
mysql> CREATE TABLE `t_manager` (
    .....
    -> `CREATE_DATETIME` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    -> `MODIFIER` varchar(32) DEFAULT NULL COMMENT '更新人',
    -> `MODIFY_DATETIME` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00' ON UPDATE CURRENT_TIMESTAMP COMMENT '修改时间',
    -> `IS_DELETED` bit(1) DEFAULT b'0' COMMENT '删除状态 1：删除 0:未删除',
    -> `IS_ENABLE` bit(1) DEFAULT b'1' COMMENT '启用状态 1:启用 0:禁用',
    -> PRIMARY KEY (`CACHE_ID`)
    -> ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
ERROR 1067 (42000): Invalid default value for 'MODIFY_DATETIME'
```
错误提示 `MODIFY_DATETIME` 字段设置的默认值是无效的，考虑到刚从5.6版本升级到5.7版本，于是又去翻了翻5.7中默认的 `sql_mode` 值。结果发现了两个可能存在影响的选项：
* `NO_ZERO_DATE` :MySQL中插入的时间字段值，不允许日期为零
* `NO_ZERO_IN_DATE` ：MySQL中插入的时间字段值，不允许日期和月份为零
#### 排错阶段二
于是解决方案就是按照 `NO_ZERO_DATE` 以及 `NO_ZERO_IN_DATE` 的要求设置默认值，将 `MODIFY_DATETIME` 字段默认值设置为'1001-01-01 01:01:01'，结果发现还是无法成功创建表：
```sql
mysql>CREATE TABLE `t_manager` (
    .....
    -> `CREATE_DATETIME` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    -> `MODIFIER` varchar(32) DEFAULT NULL COMMENT '更新人',
    -> `MODIFY_DATETIME` timestamp NOT NULL DEFAULT '1001-01-01 01:01:01' ON UPDATE CURRENT_TIMESTAMP COMMENT '修改时间',
    -> `IS_DELETED` bit(1) DEFAULT b'0' COMMENT '删除状态 1：删除 0:未删除',
    -> `IS_ENABLE` bit(1) DEFAULT b'1' COMMENT '启用状态 1:启用 0:禁用',
    -> PRIMARY KEY (`CACHE_ID`)
    -> ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
ERROR 1067 (42000): Invalid default value for 'MODIFY_DATETIME'
```

查看了所有的 `sql_mode` 值，都符合规范，但是表还是创建不成功。只好去官方手册上找找timestamp介绍：
>The TIMESTAMP data type is used for values that contain both date and time parts. TIMESTAMP has a range of '1970-01-01 00:00:01' UTC to '2038-01-19 03:14:07' UTC.

#### 排错阶段三
可以看到官方定义中timestamp字段值的范围是'1970-01-01 00:00:01'到'2038-01-19 03：14：07'，原来是我们设置的默认值不在timestamp范围之内。于是再次修改默认值：
```sql
mysql>CREATE TABLE `t_manager` (
    .....
    -> `CREATE_DATETIME` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    -> `MODIFIER` varchar(32) DEFAULT NULL COMMENT '更新人',
    -> `MODIFY_DATETIME` timestamp NOT NULL DEFAULT '1970-01-01 00:00:01' ON UPDATE CURRENT_TIMESTAMP COMMENT '修改时间',
    -> `IS_DELETED` bit(1) DEFAULT b'0' COMMENT '删除状态 1：删除 0:未删除',
    -> `IS_ENABLE` bit(1) DEFAULT b'1' COMMENT '启用状态 1:启用 0:禁用',
    -> PRIMARY KEY (`CACHE_ID`)
    -> ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
ERROR 1067 (42000): Invalid default value for 'MODIFY_DATETIME'
```
邪了门，居然还是无法成功创建表。实在是没辙了，向同事求救，同事说他在机器上试试，结果同样的语句在他的MySQL上执行成功，同样是5.7.23版本。
百思不得其解。
一气之下将两边的参数值拿出来对比了一下，果然找到了不同的根本。

| 测试环境 | 同事环境 |
| ------- | ------- |
| system_time_zone=CST | system_time_zone UTC |
| time_zone='+08:00' | time_zone=SYSTEM |

回过头来看timestamp字段定义的范围：
>The TIMESTAMP data type is used for values that contain both date and time parts. TIMESTAMP has a range of '1970-01-01 00:00:01' UTC to '2038-01-19 03:14:07' UTC.

这个时间范围指的是UTC时区的时间范围，测试环境设置了CST东八区的时区，则对应的时间范围上也需要对应的加8小时。所以将timestamp字段默认值修改为'1970-01-01 08:00:01'，表终于创建成功。
```sql
mysql>CREATE TABLE `mn_cache_refresh_manager` (
   ......
    -> `CREATE_DATETIME` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    -> `MODIFIER` varchar(32) DEFAULT NULL COMMENT '更新人',
    -> `MODIFY_DATETIME` timestamp NOT NULL DEFAULT '1970-01-01 08:00:01' ON UPDATE CURRENT_TIMESTAMP COMMENT '修改时间',
    -> `IS_DELETED` bit(1) DEFAULT b'0' COMMENT '删除状态 1：删除 0:未删除',
    -> `IS_ENABLE` bit(1) DEFAULT b'1' COMMENT '启用状态 1:启用 0:禁用',
    -> PRIMARY KEY (`CACHE_ID`)
    -> ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
Query OK, 0 rows affected (0.02 sec)
```
### 解决方案
* 将timestamp字段默认值修改为对应CST时区的最小值'1970-01-01 08:00:01'
## 总结
最后总结一下MySQL中5.7中sql_mode参数默认值的几个意思
* ONLY_FULL_GROUP_BY 
对于使用 `GROUP BY` 进行查询的SQL，不允许 `SELECT` 部分出现 `GROUP BY` 中未出现的字段，也就是 `SELECT` 查询的字段必须是 `GROUP BY` 中出现的或者使用聚合函数的或者是具有唯一属性的。
* STRICT_TRANS_TABLES 
该选项针对事务性存储引擎生效，对于非事务性存储引擎无效，该选项表示开启strict sql模式。在strict sql模式下，在INSERT或者UPDATE语句中，插入或者更新了某个不符合规定的字段值，则会直接报错中断操作
* NO_ZERO_IN_DATE 
MySQL中插入的时间字段值，不允许日期和月份为零
* NO_ZERO_DATE 
MySQL中插入的时间字段值，不允许日期为零
* ERROR_FOR_DIVISION_BY_ZERO 
INSERT或者UPDATE语句中，如果数据被0除，则出现警告（非strict sql模式下）或者错误(strict sql模式下)。
 * 当该选项关闭时，数字被0除，得到NULL且不会产生警告
 * 当该选项开启且处于非strict sql模式下，数字被0除，得到NULL但是会产生警告
 * 当该选项开启且处于strict sql模式下，数字被0除，产生错误且中断操作
* NO_AUTO_CREATE_USER 
之前版本中使用GRANT语法，如果用户不存在则会自动创建用户，该选项限制该功能
* NO_ENGINE_SUBSTITUTION 
在使用CREATE TABLE或者ALTER TABLE语法执行存储引擎的时候，如果设定的存储引擎被禁用或者未编译，会产生错误。











> 博客地址：https://win-man.github.io/  
> 公众号：欢迎关注  

![](https://user-gold-cdn.xitu.io/2018/8/16/165435ce71d2b88b?w=258&h=258&f=jpeg&s=26568)
