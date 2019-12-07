---
title: MySQL Binlog(七)——MySQL常见binlog event解析（下）
date: 2019-12-07 16:44:13
tags:
- MySQL
- Binlog
categories:
- MySQL
---

# MySQL Binlog(七)——MySQL常见binlog event解析（下）

## 前言

本文继续介绍binlog中常见event类型：Gtid_log_event、Previous_gtid_log_event、Anonymous_gtid_log_event。

## Gtid_log_event

开启GTID模式的场景下，每次事务commit提交时，MySQL会在binlog中事务开始写入一个Gtid_log_event，记录该事务的GTID事务号。

### post-header部分

| 字段 | 字节数 | 描述 |
| ----- | -------- | ----- |
| commit_flag | 1字节 | 标记事务是否提交，非0为已提交，为0表示，事务未提交 |
| sid | 16字节 | 记录GTID中uuid的部分（不记录‘-’），每1个字节表示uuid中2个字符 |
| gno | 8字节 | 小端存储，GTID中的事务号部分 |

### event-body部分

| 字段 | 字节数 | 描述 |
| ----- | -------- | ----- |
| checksum | 4字节 | 校验码 |

### 字节解析示例

使用mysqlbinlog工具解析binlog文件

```
# at 191
#180109 18:31:08 server id 330619  end_log_pos 239 CRC32 0xd20330e8    GTID [commit=yes]
SET @@SESSION.GTID_NEXT= '89fbcea2-da65-11e7-a851-fa163e618bac:5'/*!*/;
# at 239
```

解析二进制文件：

```
公共头部部分省略。。。。
01 //commit_flag:标记事务是否已提交，表示已提交
89 fb ce a2 da 65 11 e7 a8 51 fa 16 3e 61 8b ac //sid:GTID中uuid部分，转换为uuid为89fbcea2-da65-11e7-a851-fa163e618bac
05 00 00 00 00 00 00 00 //gno:GTID中的事务号，小端存储，转换为10进制为5
e8 30 03 d2 //checksum
```

## Previous_gtid_log_event

在开启GTID的模式下，每个binlog文件开始会记录一个Previous_gtid_log_event，Previous_gtid_log_event中会包含当前binlog之前所有binlog中的GTID集合。

### post-header部分

| 字段 | 字节数 | 描述 |
| ----- | -------- | ----- |
| n_sids | 8字节 | 小端存储，记录之后有几个GTID中的uuid号 |
| sid | 16字节 | 记录GTID中uuid的部分（不记录‘-’），每1个字节表示uuid中2个字符 |
| n_intervals | 8字节 |记录每个sid对应有几个间隔，指的是事务号间隔 |
| start |  8字节 |每个事务号间隔中的起始事务号 |
| end | 8字节 | 每个事务号间隔中的结束事务号+1 |

### event-body部分

| 字段 | 字节数 | 描述 |
| ----- | -------- | ----- |
| checksum | 4字节 | 校验码 |

### 字节解析示例

使用mysqlbinlog工具解析binlog文件

```
# at 120
#180111 14:10:27 server id 330619  end_log_pos 279 CRC32 0x94a92478    Previous-GTIDs
# 89fbcea2-da65-11e7-a851-fa163e618bac:1-5:999:1050-1052,
# aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa:1-2:5-7
DELIMITER ;
```

解析二进制文件：

```
公共头部部分省略。。。
02 00 00 00 00 00 00 00 //n_sids:记录之后有几个GTID中的uuid号，小端存储，转换为10进制为2，表后后续有2个uuid号
89 fb ce a2 da 65 11 e7 a8 51 fa 16 3e 61 8b ac //sid:GTID中的uuid号转换为uuid为89fbcea2-da65-11e7-a851-fa163e618bac
03 00 00 00 00 00 00 00 //n_intervals:对应的sid中中对应几个事务号间隔，小端存储，转换为10进制为3，表示有3个事务号间隔
01 00 00 00 00 00 00 00 //start:事务号间隔中的起始事务号，小端存储，转换为10进制为1
06 00 00 00 00 00 00 00 //end:事务号间隔中的结束事务号+1，小端存储，转换为10进制为6，实际的间隔结束事务号为5
e7 03 00 00 00 00 00 00 //start:事务号间隔中的起始事务号，小端存储，转换为10进制为999
e8 03 00 00 00 00 00 00 //end:事务号间隔中的结束事务号+1，小端存储，转换为10进制为1000，实际的间隔结束事务号为999
1a 04 00 00 00 00 00 00 //start:事务号间隔中的起始事务号，小端存储，转换为10进制为1050
1d 04 00 00 00 00 00 00 //end:事务号间隔中的结束事务号+1，小端存储，转换为10进制为1053，实际的间隔结束事务号为1052
aa aa aa aa aa aa aa aa aa aa aa aa aa aa aa aa //sid:GTID中的uuid号转换为uuid为aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa
02 00 00 00 00 00 00 00 //n_intervals:对应的sid中中对应几个事务号间隔，小端存储，转换为10进制为2，表示有2个事务号间隔
01 00 00 00 00 00 00 00 //start:事务号间隔中的起始事务号，小端存储，转换为10进制为1
03 00 00 00 00 00 00 00 //end:事务号间隔中的结束事务号+1，小端存储，转换为10进制为3，实际的间隔结束事务号为2
05 00 00 00 00 00 00 00 //start:事务号间隔中的起始事务号，小端存储，转换为10进制为5
08 00 00 00 00 00 00 00 //end:事务号间隔中的结束事务号+1，小端存储，转换为10进制为8，实际的间隔结束事务号为7
78 24 a9 94 //checksum
```

## Anonymous_gtid_log_event

MySQL在binlog中记录每一个匿名事务之前会记录一个Anonymous_gtid_log_event表示接下来的事务是一个匿名事务。
**注意：因为在5.6.34中并不会产生Anonymous_gtid_log_event，所以以下内容是基于5.7.19版本解析**

### post-header部分

| 字段 | 字节数 | 描述 |
| ----- | -------- | ----- |
| gtid_flags | 1字节 | 记录binlog格式，如果gtid_flags值为1，表示binlog中可能有以statement方式记录的binlog，如果为0表示，binlog中只有以row格式记录的binlog |
| sid | 16字节 | 记录GTID中uuid的部分（不记录‘-’），每1个字节表示uuid中2个字符 |
| gno | 8字节 | 小端存储，GTID中的事务号部分 |
| logical_timestamp_typecode | 1字节 | 判断是否有last_commit和sequence_no，在logical_tmiestamp_typecode=2的情况下，有last_commit和sequence_no |
| last_commit | 8字节 | 小端存储，上次提交的事务号 |
| sequence_no | 8字节 | 小端存储，本次提交的序列号 |

### event-body部分

| 字段 | 字节数 | 描述 |
| ----- | -------- | ----- |
| checksum | 4字节 | 校验码 |

### 字节解析示例

使用mysqlbinlog工具解析binlog文件

```
#180109 10:53:54 server id 9999  end_log_pos 5681 CRC32 0x46be0639    Anonymous_GTID    last_committed=20    sequence_number=21    rbr_only=yes
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 5681
```

解析二进制文件：

```
公共头部部分省略。。。
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 //sid:
00 00 00 00 00 00 00 00 //gno:
02 //logical_timestamp_typecode:判断是否有last_commit和sequence_no，在logical_tmiestamp_typecode=2的情况下，有last_commit和sequence_no，所以这边的话，后续有last_commit和sequence_no部分的内容
14 00 00 00 00 00 00 00 //last_commit:上次提交的事务号，小端存储，转换为10机制为20
15 00 00 00 00 00 00 00 //sequence_no:本次提交的序列号，小端存储，转换为10进制为21
39 06 be 46 //checksum:校验码
```


