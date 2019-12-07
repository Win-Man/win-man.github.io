---
title: MySQL Binlog(六)——MySQL常见binlog event解析（中）
date: 2019-12-07 16:43:13
tags:
- MySQL
- Binlog
categories:
- MySQL
---

# MySQL Binlog(六)——MySQL常见binlog event解析（中）

## 前言

本文继续介绍binlog中常见的event：Table_map_event、Write_rows_log_event、Update_rows_log_event、Delete_rows_log_event、Xid_event。

## Table_map_event

TABLE_MAP_EVENT只有在binlog文件是以ROW格式记录的时候，才会使用。binlog中记录的每个更改的记录之前都会有一个对应要操作的表的TABLE_MAP_EVENT。TABLE_MAP_EVENT中记录了表的定义（包括database name,table name，字段定义），并且会将这个表的定义对应于一个数字，称为table_id。设计TABLE_MAP_EVENT类型event的目的是为了当主库和从库之间有不同的表定义的时候，复制仍能进行。如果一个事务中操作了多个表，多行记录，在binlog中会将对多行记录的操作event进行分组，每组行记录操作event前面会出现对应表的TABLE_MAP_EVENT。

### post-header部分

| 字段 | 字节数 | 描述 |
| ----- | -------- | ----- |
| table_id | 6字节 | 操作的表的table_id |
| flags | 2字节 | 目前版本没有用，都是0，保留给之后的版本使用 |

### event-body部分

| 字段 | 字节数 | 描述 |
| ----- | ------- | ------ |
| database_name | 1个字节表示字符串长度，之后接一个null-terminated类型的字符串 | | 操作的数据库的名称 |
| table_name | 1个字节表示字符串长度，之后接一个null-terminated类型的字符串 | 操作的表的名称 |
| column_count | packed integer(1或3或4或9个字节) | 对应表中的字段数量 |
| column_type | 每个字段占用一个字节，字段类型定义在enum_field_types中 | 字段类型 |
| metadata_length| packed integer(1或3或4或9个字节)  | 对应字段的元数据信息的长度  |
| metadata |  根据enum_field_type中的定义确定不同字段的元数据，如果某字段类型没有元数据，则不记录 |每个字段的元数据信息，比如varchar字段需要记录最长长度 |
| null_bits | int((column_count + 7)/8) 字节 | 一个bit表示一个字段是否可以为NULL，顺序是：第一个字节的最低位开始向最高位增长，之后第二个字节的最低位开始向最高位增长，以此类推 |
| check_sum | 4字节 | 校验码 |



### 字节解析示例

建表语句:

```
CREATE TABLE `test1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(20) COLLATE utf8_bin DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=20 DEFAULT CHARSET=utf8 COLLATE=utf8_bin
```

二进制内容解析：

```
公共头部部分省略
6c 00 00 00 00 00 //table_id :小端存储，16进制转换为10进制为108
01 00 //flag :
08 67 61 6e 67 73 68 65 6e 00 //database_name :1个字节是字符串长度，后接一个null-terminated string 第一个字节表示字符串长度为8，后面内容将16进制转换为ascii字符为gangshen
05 74 65 73 74 31 00 //table_name: 1个字节是字符串长度，后接一个null-terminated string第一个字节表示字符串长度为5，后面内容将16进制转换为ascii字符为test1
02 //columns count :packet integer类型，转换之后，数值为2 表示表中有两个字段
03 0f //column type : 一个字节表示一个字段的类型，字段类型定义在enum enum_field_types ，分别是一个int类型以及一个varchar类型,具体的字段类型及记录格式等信息可以查看《MySQL中Rows_event中字段表示》文档
02 //metadata length : packet integer类型,转换之后，数值为2，表示记录表中的metadata内容占用2个字节
14 00 // :varchar 的max length 因为int没有metadata所以跳过
02 //null_bits :int((column_count + 7) / 8)个字节 一个bit表示一个字段是否可以为null 
52 53 4a d9 checksum
```

## Write_rows_log_event

在以ROW格式记录的binlog文件中，Write_rows_log_event记录了插入的行记录。

### post-header部分

| 字段 | 字节数 | 描述 |
| ----- | -------- | ----- |
| table_id | 6字节 | 操作的表的table_id |
| flags | 2字节 | 目前版本没有用，都是0，保留给之后的版本使用 |

### event-body部分

| 字段 | 字节数 | 描述 |
| ----- | -------- | ----- |
| var_header | 2字节 | |
| m_width | packed integer(1或3或4或9个字节) | 表中字段数量 |
| before_image | (m_width + 7)/8字节 | |
| after_bitmap_bits | (m_with +7)/8字节 | 字段值是否为空标记，一个bit表示一个字段是否为NULL，顺序是：第一个字节的最低位开始向最高位增长，之后第二个字节的最低位开始向最高位增长，以此类推 |
| after_column_content | 不定 | 按照顺序每个字段的内容 |
| check_sum | 4字节 | 校验码 |

### 字节解析示例

建表语句

```
CREATE TABLE `int_table` (
  `col1` tinyint(4) DEFAULT NULL,
  `col2` smallint(6) DEFAULT NULL,
  `col3` mediumint(9) DEFAULT NULL,
  `col4` int(11) DEFAULT NULL,
  `col5` bigint(20) DEFAULT NULL,
  `col6` tinyint(1) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin
```

插入数据:

```
insert into int_table values(1,11,111,1111,11111,true);
```

对应Write_rows_log_event使用mysqlbinlog解析

```
# at 340
#180103 10:21:20 server id 330619  end_log_pos 395 CRC32 0x50177e10    Write_rows: table id 100 flags: STMT_END_F
### INSERT INTO `gangshen`.`int_table`
### SET
###  @1=1 /* TINYINT meta=0 nullable=1 is_null=0 */
###  @2=11 /* SHORTINT meta=0 nullable=1 is_null=0 */
###  @3=111 /* MEDIUMINT meta=0 nullable=1 is_null=0 */
###  @4=1111 /* INT meta=0 nullable=1 is_null=0 */
###  @5=11111 /* LONGINT meta=0 nullable=1 is_null=0 */
###  @6=1 /* TINYINT meta=0 nullable=1 is_null=0 */
# at 395
```

解析二进制文件:

```
公共头部部分省略
64 00 00 00 00 00 //table_id: 小端存储，16进制转换为10进制为100
01 00 //flag:
02 00 //var header length :小端存储，16进制转换为10进制为2
06 //m_width :packet integer，表示表中字段数量
ff //before image: (m_width + 7) / 8字节
c0 //after_bitmap_bits :insert插入之后的记录的NULL标记，表中六个字段，插入的值都不为NULL
//after_columns_content：insert插入记录之后的记录值
01 
0b 00 
6f 00 00 
57 04 00 00
67 2b 00 00 00 00 00 00
01 
10 7e 17 50 //checksum
```



## Update_rows_log_event

在以ROW格式记录的binlog文件中，Update_rows_log_event记录了更新的行记录。

### post-header部分

| 字段 | 字节数 | 描述 |
| ----- | -------- | ----- |
| table_id | 6字节 | 操作的表的table_id |
| flags | 2字节 | 目前版本没有用，都是0，保留给之后的版本使用 |



### event-body部分

| 字段 | 字节数 | 描述 |
| ----- | -------- | ----- |
| var_header | 2字节 | |
| m_width | packed integer(1或3或4或9个字节) | 表中字段数量 |
| before_image | (m_with+7)/8字节 | |
| after_image | (m_with+7)/8字节 | |
| before_bitmap_bits | (m_with+7)/8字节 | before_bitmap_bits记录的是update更改之前的记录中，字段值是否为NULL标记，一个bit表示一个字段是否为NULL，顺序是：第一个字节的最低位开始向最高位开始增长，之后第二个字节的最低位开始向最高位增长，以此类推 |
| before_column_content | 不定 | update更新之前的记录值，按照顺序每个字段的内容 |
| after_bitmap_bits | (m_with+7)/8字节 | after_bitmap_bits记录的是update更改之前的记录中，字段值是否为NULL标记，一个bit表示一个字段是否为NULL，顺序是：第一个字节的最低位开始向最高位开始增长，之后第二个字节的最低位开始向最高位增长，以此类推 |
| after_column_content | 不定 | update更新之后的记录值，按照顺序每个字段的内容 |
| check_sum | 4字节 | 校验码 |



### 字节解析示例

建表语句

```
CREATE TABLE `int_table` (
  `col1` tinyint(4) DEFAULT NULL,
  `col2` smallint(6) DEFAULT NULL,
  `col3` mediumint(9) DEFAULT NULL,
  `col4` int(11) DEFAULT NULL,
  `col5` bigint(20) DEFAULT NULL,
  `col6` tinyint(1) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin
```

插入数据并更新:

```
root@localhost : gangshen 10:20:49> insert into int_table values(1,11,111,1111,11111,true);
Query OK, 1 row affected (0.03 sec)

root@localhost : gangshen 10:21:20> update int_table set col2=22,col3=222 where col1=1;
Query OK, 1 row affected (0.02 sec)
Rows matched: 1  Changed: 1  Warnings: 0

root@localhost : gangshen 10:21:44> select * from int_table;
+------+------+------+------+-------+------+
| col1 | col2 | col3 | col4 | col5  | col6 |
+------+------+------+------+-------+------+
|    1 |  22 |  222 | 1111 | 11111 |    1 |
+------+------+------+------+-------+------+
1 row in set (0.00 sec)
```

对应Update_rows_log_event使用mysqlbinlog解析

```
# at 644
#180103 10:41:37 server id 9999  end_log_pos 720 CRC32 0x5576b52f    Update_rows: table id 231 flags: STMT_END_F
### UPDATE `gangshen`.`int_table`
### WHERE
###  @1=1 /* TINYINT meta=0 nullable=1 is_null=0 */
###  @2=11 /* SHORTINT meta=0 nullable=1 is_null=0 */
###  @3=111 /* MEDIUMINT meta=0 nullable=1 is_null=0 */
###  @4=1111 /* INT meta=0 nullable=1 is_null=0 */
###  @5=11111 /* LONGINT meta=0 nullable=1 is_null=0 */
###  @6=1 /* TINYINT meta=0 nullable=1 is_null=0 */
### SET
###  @1=1 /* TINYINT meta=0 nullable=1 is_null=0 */
###  @2=22 /* SHORTINT meta=0 nullable=1 is_null=0 */
###  @3=222 /* MEDIUMINT meta=0 nullable=1 is_null=0 */
###  @4=1111 /* INT meta=0 nullable=1 is_null=0 */
###  @5=11111 /* LONGINT meta=0 nullable=1 is_null=0 */
###  @6=1 /* TINYINT meta=0 nullable=1 is_null=0 */
# at 720
```

解析二进制文件:

```
公共头部部分省略。。。
64 00 00 00 00 00 //table_id: 小端存储，16进制转换为10进制为100
01 00 //flag:
02 00 //var header length :小端存储，16进制转换为10进制为2
06 //m_width :packet integer，表示表中字段数量
ff //before image: (m_width + 7) / 8字节
ff //after image: (m_width + 7) / 8字节
c0 //before_bitmap_bits :update更新之前记录中NULL标记，表中六个字段，值都不为NULL
//before_column_content接下来是update更新之前记录的值
01 
0b 00
6f 00 00 
57 04 00 00
67 2b 00 00 00 00 00 00
01 
c0 //after_bitmap_bits :update更新之后记录中NULL标记，表中六个字段，值都不为NULL
//after_column_content接下来是update更新之前记录的值
01 
16 00 
de 00 00 
57 04 00 00
67 2b 00 00 00 00 00 00 
01 
16 a3 de bb //checksum
```

## Delete_rows_log_event

在以ROW格式记录的binlog文件中，Delete_rows_log_event记录了删除的行记录。

### post-header部分

| 字段 | 字节数 | 描述 |
| ----- | -------- | ----- |
| table_id | 6字节 | 操作的表的table_id |
| flags | 2字节 | 目前版本没有用，都是0，保留给之后的版本使用 |

### event-body部分

| 字段 | 字节数 | 描述 |
| ----- | -------- | ----- |
| var_header | 2字节 | |
| m_width | packed integer(1或3或4或9个字节) | 表中字段数量 |
| after_image | (m_width + 7)/8字节 | |
| before_bitmap_bits | (m_with+7)/8字节 | before_bitmap_bits记录的是update更改之前的记录中，字段值是否为NULL标记，一个bit表示一个字段是否为NULL，顺序是：第一个字节的最低位开始向最高位开始增长，之后第二个字节的最低位开始向最高位增长，以此类推 |
| before_column_content | 不定 | delete删除之前的记录值，按照顺序每个字段的内容 |
| check_sum | 4字节 | 校验码 |

### 字节解析示例

建表语句

```
CREATE TABLE `int_table` (
  `col1` tinyint(4) DEFAULT NULL,
  `col2` smallint(6) DEFAULT NULL,
  `col3` mediumint(9) DEFAULT NULL,
  `col4` int(11) DEFAULT NULL,
  `col5` bigint(20) DEFAULT NULL,
  `col6` tinyint(1) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin
```

删除数据:

```
delete from int_table where col1=1;
```

对应Delete_rows_log_event使用mysqlbinlog解析

```
# at 320
#180103 14:24:54 server id 330619  end_log_pos 375 CRC32 0xce4818b2    Delete_rows: table id 100 flags: STMT_END_F
### DELETE FROM `gangshen`.`int_table`
### WHERE
###  @1=1 /* TINYINT meta=0 nullable=1 is_null=0 */
###  @2=22 /* SHORTINT meta=0 nullable=1 is_null=0 */
###  @3=222 /* MEDIUMINT meta=0 nullable=1 is_null=0 */
###  @4=1111 /* INT meta=0 nullable=1 is_null=0 */
###  @5=11111 /* LONGINT meta=0 nullable=1 is_null=0 */
###  @6=1 /* TINYINT meta=0 nullable=1 is_null=0 */
# at 375
```

解析二进制文件:

```
公共头部部分省略
64 00 00 00 00 00 //table_id: 小端存储，16进制转换为10进制为100
01 00 //flag:
02 00 //var header length :小端存储，16进制转换为10进制为2
06 //m_width :packet integer，表示表中字段数量
ff //after image: (m_width + 7) / 8字节
c0 //before_bitmap_bits :delete删除之前的记录的NULL标记，表中六个字段，值都不为NULL
//before_columns_content：delete删除记录之前的记录值
01 
16 00 
de 00 00 
57 04 00 00
67 2b 00 00 00 00 00 00
01 
10 7e 17 50 //checksum
```

## Xid_event

表示支持内部XA的存储引擎上的事务提交。正常的事务是通过`QUERY_EVENT`来发送一个`BEGIN`语句并且通过`QUERY_EVENT`来发送一个`COMMIT`语句（如果事务回滚则发送的是`ROLLBACK`）实现。

### post-header部分

无

### event-body部分

| 字段 | 字节数 | 描述 |
| ----- | -------- | ----- |
| xid | 8字节 | xid号，小端存储，无符号数 |
| check_sum | 4字节 | 校验码 |

### 字节解析示例

使用mysqlbinlog工具解析对应的Xid_event

```
# at 1691
#180103 15:30:45 server id 330619  end_log_pos 1722 CRC32 0x8816181c    Xid = 2698
COMMIT/*!*/;
# at 1722
```

解析二进制文件：

```
公共头部部分省略...
8a 0a 00 00 00 00 00 00 //xid,因为是小端存储，所以实际为0x 00 00 00 00 00 00 0a 8a，转换为10进制为2698
1c 18 16 88 //check_sum
```


