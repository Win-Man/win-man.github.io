---
title: MySQL Binlog(四)—— MySQL binlog event解析——基础知识
date: 2019-12-07 16:41:13
tags:
- MySQL
- Binlog
categories:
- MySQL
---

# MySQL Binlog(四)—— MySQL binlog event解析——基础知识

## 前言

MySQL复制实质上就是讲binlog同步到从库上进行应用，前面的文章已经讲了如何建立复制连接，从这篇文章开始，我们来讲讲binlog里面的内容，看看主从之间同步具体同步了哪些内容，首先会介绍一些关于binlog event的基础知识，之后的文章中会详细对binlog event进行分析。

## event基础知识

### event类型

5.6.34版本的MySQL的event类型有：

```
enum Log_event_type
{ UNKNOWN_EVENT= 0,
  START_EVENT_V3= 1,
  QUERY_EVENT= 2,
  STOP_EVENT= 3,
  ROTATE_EVENT= 4,
  INTVAR_EVENT= 5,
  LOAD_EVENT= 6,
  SLAVE_EVENT= 7,  
  CREATE_FILE_EVENT= 8,
  APPEND_BLOCK_EVENT= 9,
  EXEC_LOAD_EVENT= 10,
  DELETE_FILE_EVENT= 11,
  NEW_LOAD_EVENT= 12,
  RAND_EVENT= 13,
  USER_VAR_EVENT= 14,
  FORMAT_DESCRIPTION_EVENT= 15,
  XID_EVENT= 16,
  BEGIN_LOAD_QUERY_EVENT= 17,
  EXECUTE_LOAD_QUERY_EVENT= 18,
  TABLE_MAP_EVENT = 19,
  PRE_GA_WRITE_ROWS_EVENT = 20,
  PRE_GA_UPDATE_ROWS_EVENT = 21,
  PRE_GA_DELETE_ROWS_EVENT = 22,
  WRITE_ROWS_EVENT_V1 = 23,
  UPDATE_ROWS_EVENT_V1 = 24,
  DELETE_ROWS_EVENT_V1 = 25,
  INCIDENT_EVENT= 26,
  HEARTBEAT_LOG_EVENT= 27,
  IGNORABLE_LOG_EVENT= 28,
  ROWS_QUERY_LOG_EVENT= 29,
  WRITE_ROWS_EVENT = 30,
  UPDATE_ROWS_EVENT = 31,
  DELETE_ROWS_EVENT = 32,
  GTID_LOG_EVENT= 33,
  ANONYMOUS_GTID_LOG_EVENT= 34,
  PREVIOUS_GTIDS_LOG_EVENT= 35,
  ENUM_END_EVENT /* end marker */
};
```

### event类型含义

这边话介绍几个常见的event类型所对应的含义，对于不常见或者现在基本上已经不用了的event类型，这边不做过多的描述，想了解详细的内容，可以查看链接:
[https://dev.mysql.com/doc/internals/en/event-meanings.html](https://dev.mysql.com/doc/internals/en/event-meanings.html)

* FORMAT_DESCRIPTION_EVENT

FORMAT_DESCRIPTION_EVENT是每个binlog文件开头的一个event。

* ROTATE_EVENT

当MySQL切换至新的binlog文件的时候，MySQL会在旧的binlog文件中写入一个ROTATE_EVENT，表示新的binlog文件的文件名，以及第一个偏移地址。当在数据库中执行`FLUSH LOGS`语句或者binlog文件的大小超过max_binlog_size就会切换新的binlog文件。 

* TABLE_MAP_EVENT

TABLE_MAP_EVENT只有在binlog文件是以ROW格式记录的时候，才会使用。binlog中记录的每个更改的记录之前都会有一个对应要操作的表的TABLE_MAP_EVENT。TABLE_MAP_EVENT中记录了表的定义（包括database name,table name，字段定义），并且会将这个表的定义对应于一个数字，称为table_id。设计TABLE_MAP_EVENT类型event的目的是为了当主库和从库之间有不同的表定义的时候，复制仍能进行。如果一个事务中操作了多个表，多行记录，在binlog中会将对多行记录的操作event进行分组，每组行记录操作event前面会出现对应表的TABLE_MAP_EVENT。

* QUERY_EVENT

记录更新操作的语句。

* WRITE_ROWS_EVENT

在以ROW格式记录的binlog文件中，WRITE_ROWS_EVENT记录了插入的行记录。

* UPDATE_ROWS_EVENT

在以ROW格式记录的binlog文件中，UPDATE_ROWS_EVENT记录了更新的行记录。

* DELETE_ROWS_EVENT

在以ROW格式记录的binlog文件中，DELETE_ROWS_EVENT记录了删除的行记录。

## event字节解析

在MySQL的发展过程中，一共出现过三个版本的event结构：

* v1:在MySQL 3.23中使用
* v2:在MySQL4.0.2至4.1中使用
* v4:在MySQL5.0及以上版本中使用

因为目前MySQL数据库版本基本上都是5.0版本以上的，所以本文就只介绍v4版本的event架构。

### event组织架构

v4版本的event主要由event header部分和post-header以及variable part部分组成。

* event header是每个event都会有的部分，并且每个event header的格式是相同的，固定为19个字节长度，在event header 中记录了event_length、type_code等信息，可以表示event的长度，event的类型，下一个event的偏移位置等信息，都可以再event header中获取到。
* extra_headers指定了除公共头部外的内容，但是目前版本的binlog格式中，这一部分不存在的
* fixed_part部分在某些文章中也被称做post-header,每个类型的event之间的post-header部分是不相同的，但是同一类型的event占用的post-header

字节数是一样的。每个类型的event占用多少字节为post-header，这个定义在了Format_description_event中，下面讲Format_description_event的章节中会介绍。

* variable part是event真实记录具体信息的部分。

event字节结构表示如下图：

`x:y`中x表示该部分从多少字节偏移开始，y表示该部分占用多少字节。
* timestamp记录的是该事务记录的时间
* type_code，记录的是该event的类型，具体的event类型，见上面的Log_event_type
* server_id，记录的是执行该event的server的server_id号
* event_length，该event的长度，共占多少字节
* next_position，下一个event在binlog文件中的偏移位置
* flag:标记
* extra_headers，额外的头部内容，现在是空的，不存在
* fixed part有些时候也被称作post-header
* variable part有些时候也被称作payload或者body



### event公共头部

从上述的event结构中，可以看到，每个event的header部分的内容结构是一致的，所有的event，都会记录这些信息，这是所有的event的公共头部。所以我们把这块内容单独拿出来讲。

| 字段 | 字节数 | 描述 |

| ----- | -------- | ----- |

| timestamp | 4字节 | 时间戳 |

| type_code | 1字节 | 记录event类型 |

| server_id | 4 字节 | 记录event生成的MySQL所对应的server_id |

| event_length | 4字节 | 这个event的字节长度，包括event header |

| next_position | 4字节 | 下一个event在binlog文件中的偏移位置 |

| flags | 2字节 | |



示例:我们拿一个Write_rows_event的字节内容作为示例，解析event内容:

```

b6 1e 1c 5a //timestamp:小端存储，将16进制转换成10进制为1511792310，将时间戳转换为时间为2017-11-27 22:18:30

1e //type_code:event类型，0x1e转换为10进制为30，查看Log_event_type中，看到30的为WRITE_ROWS_EVENT

0f 27 00 00 //server_id :小端存储，将16进制转换为10进制为9999

31 00 00 00 //event_length :小端存储，将16进制转换为10进制为49

b1 03 00 00 //next_position:小端存储，将16进制转换为10进制为945

00 00 //flags:

body部分省略

```



## 参考链接

* https://dev.mysql.com/doc/internals/en/event-structure.html

* https://dev.mysql.com/doc/internals/en/event-meanings.html


