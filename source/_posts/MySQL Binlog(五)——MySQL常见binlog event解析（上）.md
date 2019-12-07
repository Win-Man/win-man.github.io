---
title: MySQL Binlog(五)——MySQL常见binlog event解析（上）
date: 2019-12-07 16:42:13
tags:
- MySQL
- Binlog
categories:
- MySQL
---

# MySQL Binlog(五)——MySQL常见binlog event解析（上）

## 前言

前面讲了binlog event的基础知识，从这篇文章开始我们就具体来解析binlog中常见的event，了解其具体代表的意义。本文会介绍Format_description_event、Rotate_event、Query_log_event、Rows_query_log_event几种event。

## Format_description_event

Format_description_event是每个binlog文件开头的一个描述信心的event。

### post-header部分

| 字段 | 字节数 | 描述 |
| ----- | -------- | ----- |
| binlog_version | 2字节 | binlog结构的版本,v1,v2,v4 |
| server_version | 50字节 | MySQL server的数据库版本 |
| timestamp | 4字节 | 创建该binlog文件的时间,单位为秒 |
| header_length | 1字节 | 指定公共头部的长度，v4版本中这个值一直为19，说明其他所有的event的extra_header部分都是空,extra_header长度为header_length-19 |
| event_post_header_length | variable size，跟各个版本支持的event类型总数一致 | 每个event类型，占用一个字节，表示该event类型post-header部分的长度 |


### event-body部分

无

### 字节解析示例
```
公共头部部分省略
04 00 //binlog_version :v4版本
35 2e 36 2e 33 34 2d 6c 6f 67 
00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 00 00  //mysql server version :16进制转换为ascii之后，值为5.6.34-log
00 00 00 00 //timestamp:
13 //header length :19,说明该版本的event类型公共头部长度都为19，extra_header长度为header_length-19
38 0d 00 08 00 12 00 04 04 04 
04 12 00 00 5c 00 04 1a 08 00 
00 00 08 08 08 02 00 00 00 0a 
0a 0a 19 19 00 //event_post_header_len:各个类型event的post-header部分长度
01 2e ef 23 e2
```


## Rotate_event

当MySQL切换至新的binlog文件的时候，MySQL会在旧的binlog文件中写入一个ROTATE_EVENT，其内容包含新的binlog文件的文件名以及第一个偏移地址。当在数据库中执行`FLUSH LOGS`语句或者binlog文件的大小超过max_binlog_size参数设定的值就会切换新的binlog文件。
### post-header部分

| 字段 | 字节数 | 描述 |
| ----- | -------- | ----- |
|next_binlog_po | 8字节 | 下一个binlog文件起始的偏移地址 |


### event-body部分

| 字段 | 字节数 | 描述 |
| ----- | -------- | ----- |
|next_binlog_filename | variable string | 下一个binlog文件的文件名 |


### 字节解析示例
```
公共头部部分省略
04 00 00 00 00 00 00 00 //next_binlog_pos：下一个binlog的起始偏移地址,小端存储，16进制转换为10进制之后为4
6d 79 73 71 6c 2d 62 69 6e 2e 30 30 30 30 30 32  //next_binlog_filename：下一个binlog的文件名,16进制转换为ascii之后，值为mysql-bin.000002
b3 db 0b 9a checksum
```

## Query_log_event

记录更新操作的语句。

### post-header部分

| 字段 | 字节数 | 描述 |
| ----- | -------- | ----- |
| thread_id | 4字节 | 小端存储，执行语句的线程ID号 |
| exec_time | 4字节 | 小端存储，语句执行的时间 |
| db_len | 1字节 | database名的长度 |
| error_code | 2字节 | 错误号 |
| status_vars_len | 2字节 | 小端存储，这部分，在v1和v3版本的event中是没有的，指定状态值的长度 |

### event-body部分
| 字段 | 字节数 | 描述 |
| ----- | -------- | ----- |
| status_vars | status_vars_len字节 | 记录状态值，具体的解析见下表 |
| database_name | db_len+1字节 | null-terminaled类型的字符串，记录database的名字 |
| query_statement | 不定 | 执行的语句 |
| check_sum | 4字节 | 校验码 |

status_vars的解析是，一个字节表示状态的类型，在类型之后按照类型不同紧接着不同字节数的状态值，状态的类型一共有：

```
enum Query_event_status_vars
  {
    Q_FLAGS2_CODE= 0,
    Q_SQL_MODE_CODE,
    Q_CATALOG_CODE,
    Q_AUTO_INCREMENT,
    Q_CHARSET_CODE,
    Q_TIME_ZONE_CODE,
    Q_CATALOG_NZ_CODE,
    Q_LC_TIME_NAMES_CODE,
    Q_CHARSET_DATABASE_CODE,
    Q_TABLE_MAP_FOR_UPDATE_CODE,
    Q_MASTER_DATA_WRITTEN_CODE,
    Q_INVOKER,
    Q_UPDATED_DB_NAMES,
    Q_MICROSECONDS,
    Q_COMMIT_TS,
    Q_COMMIT_TS2,
    Q_EXPLICIT_DEFAULTS_FOR_TIMESTAMP
  };
```

不同状态对应的状态值的字节数 

| 状态 | 对用的code | 状态值占用的字节数 |
| ----- | -------------- | ----------------------- |
| Q_FLAGS2_CODE | 0 | 4字节 |
| Q_SQL_MODE_CODE | 1 | 8字节 |
| Q_CATALOG_CODE | 2 | 第一个字节表示catalog_len，总共catalog_len+2个字节 |
| Q_AUTO_INCREMENT | 3 | 4字节 |
| Q_CHARSET_CODE | 4 | 6字节 |
| Q_TIME_ZONE_CODE | 5 | 第一个字节表示time_zone_len，总共time_zone_len+1字节 |
| Q_CATALOG_NZ_CODE | 6 | 第一个字节表示catalog_len，总共catalog_len+1个字节 |
| Q_LC_TIME_NAMES_CODE | 7 | 2字节 |
| Q_CHARSET_DATABASE_CODE | 8 | 2字节 |
| Q_TABLE_MAP_FOR_UPDATE_CODE | 9 | 8字节 |
| Q_MASTER_DATA_WRITTEN_CODE | 10 | 4字节 |
| Q_INVOKER | 11 | 包含两部分，一部分是user，一部分是host。user部分，一个字节表示user_len，接着user_len个字节表示user。host部分，一个字节表示host_len，接着host_len个字节表示host。 |
| Q_UPDATED_DB_NAMES | 12 |  |
| Q_MICROSECONDS | 13 | 3字节 |
| Q_COMMIT_TS | 14 |   |
| Q_COMMIT_TS2 | 15 | |
| Q_EXPLICIT_DEFAULTS_FOR_TIMESTAMP | 16 | 1字节 | 

### 字节解析示例

执行语句:

```
insert into test1(`name`) values('beijing');

root@localhost : (none) 06:07:04> show binlog events  in 'mysql-bin.000012';
+------------------+-----+-------------+-----------+-------------+-------------------------------------------------------------+
| Log_name        | Pos | Event_type  | Server_id | End_log_pos | Info                                                        |
+------------------+-----+-------------+-----------+-------------+-------------------------------------------------------------+
| mysql-bin.000012 |  4 | Format_desc |    330619 |        120 | Server ver: 5.6.34-log, Binlog ver: 4                      |
| mysql-bin.000012 | 120 | Query      |    330619 |        212 | BEGIN                                                      |
| mysql-bin.000012 | 212 | Intvar      |    330619 |        244 | INSERT_ID=28                                                |
| mysql-bin.000012 | 244 | Query      |    330619 |        374 | use `gangshen`; insert into test1(`name`) values('beijing') |
| mysql-bin.000012 | 374 | Xid        |    330619 |        405 | COMMIT /* xid=2961 */                                      |
+------------------+-----+-------------+-----------+-------------+-------------------------------------------------------------+
5 rows in set (0.00 sec)
```
使用mysqlbinlog工具解析binlog文件

```
# at 244
#180105 15:20:29 server id 330619  end_log_pos 244 CRC32 0xf41b5eaa    Intvar
SET INSERT_ID=28/*!*/;
#180105 15:20:29 server id 330619  end_log_pos 374 CRC32 0x98379221    Query    thread_id=106404    exec_time=0    error_code=0
use `gangshen`/*!*/;
SET TIMESTAMP=1515183629/*!*/;
insert into test1(`name`) values('beijing')
/*!*/;
# at 374
```

解析二进制文件：

```
公共头部部分省略。。。。
a4 9f 01 00 //thread_id:执行语句的线程号，小端存储，转换为16进制为106404
00 00 00 00 //exec_time:执行语句的时间，小端存储，转化为16进制为0
08 //db_len:database name的长度，转换为10进制为8
00 00 //error_code:错误号，小端存储，转换为10进制为0
2a 00 //status_vars_len:状态值的长度，小端存储，转换为10进制为42，表示后续的42个字节为状态值的内容
//status_vars_len
00 00 00 00 00 
01 00 00 20 40 00 00 00 00
06 03 73 74 64 
03 02 00 02 00
04 21 00 21 00 53 00 
0c 01 67 61 6e 67 73 68 65 6e 00 
67 61 6e 67 73 68 65 6e 00 //database_name:数据库名称，null-terminaled string类型 ，转换为字符串为gangshen
696e7365727420696e746f20746573743128606e616d6560292076616c75657328276265696a696e672729 //query_statement:执行的语句，转换为字符串为:insert into test1(`name`) values('beijing')
21 92 37 98//checksum
```

## Rows_query_log_event
在row格式复制模式下，将query以语句形式记录。在5.6.2版本之后，可以通过设置`binlog_rows_query_log_events`参数来控制在row格式复制模式下是否需要将query语句记录到binlog文件中。
### post-header部分
无

### event-body部分

| 字段 | 字节数 | 描述 |
| ----- | -------- | ----- |
| str_len |1字节 | 记录语句长度 |
| statement | str_len字节 | 对应的语句 | 
| checksum | 4字节 | 校验码 |

### 字节解析示例

执行语句:

```
root@localhost : gangshen 09:11:56> insert into test1(`name`) values('rows_query');
Query OK, 1 row affected (0.04 sec)

root@localhost : gangshen 09:12:06> show binlog events in 'mysql-bin.000014';
+------------------+-----+-------------+-----------+-------------+--------------------------------------------------+
| Log_name        | Pos | Event_type  | Server_id | End_log_pos | Info                                            |
+------------------+-----+-------------+-----------+-------------+--------------------------------------------------+
| mysql-bin.000014 |  4 | Format_desc |    330619 |        120 | Server ver: 5.6.34-log, Binlog ver: 4            |
| mysql-bin.000014 | 120 | Query      |    330619 |        201 | BEGIN                                            |
| mysql-bin.000014 | 201 | Rows_query  |    330619 |        271 | # insert into test1(`name`) values('rows_query') |
| mysql-bin.000014 | 271 | Table_map  |    330619 |        326 | table_id: 98 (gangshen.test1)                    |
| mysql-bin.000014 | 326 | Write_rows  |    330619 |        377 | table_id: 98 flags: STMT_END_F                  |
| mysql-bin.000014 | 377 | Xid        |    330619 |        408 | COMMIT /* xid=3032 */                            |
+------------------+-----+-------------+-----------+-------------+--------------------------------------------------+
6 rows in set (0.00 sec)
```

使用mysqlbinlog工具解析binlog文件

```
# at 201
#180108  9:12:06 server id 330619  end_log_pos 271 CRC32 0xdc68af8a    Rows_query
# insert into test1(`name`) values('rows_query')
# at 271
```

解析二进制文件：

```
公共头部部分省略。。。
2e //str_len:执行语句的长度，转换为10进制为46
696e7365727420696e746f20746573743128606e616d6560292076616c7565732827726f77735f71756572792729  //statement:执行的语句，转换为ascii为insert into test1(`name`) values('rows_query')
8a af 68 dc //check_sum
```

## 参考链接
* https://dev.mysql.com/doc/internals/en/event-meanings.html





