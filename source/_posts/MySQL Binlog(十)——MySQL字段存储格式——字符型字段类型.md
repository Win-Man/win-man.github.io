---
title: MySQL Binlog(十)——MySQL字段存储格式——字符型字段类型
date: 2019-12-07 16:55:13
tags:
- MySQL
- Binlog
categories:
- MySQL
---

# MySQL Binlog(十)——MySQL字段存储格式——字符型字段类型

## 前言

本文继续介绍binlog中字符类型字段的存储格式。

## 字符型字段类型

中文等字符存储以该字段的字符集规定存储二进制数据。

### CHAR和VARCHAR类型

* [NATIONAL] CHAR[( M )] [CHARACTER SET charset_name ] [COLLATE collation_name ]

定长字符串存储时总是使用空格填充右侧达到指定长度。M表示字符串列的长度。M的范围是0到255。省略的花，长度为1。

CHAR是CHARACTER的简写。NATIONAL CHAR（或简写NCHAR）是标准的定义CHAR列应该使用默认字符集的SQL方法。MySQL4.1及更高版本使用utf8作为默认字符集。

CHAR BYTE是BINARY的别名。这是为了保证兼容性。

MySQL允许定义创建一个CHAR(0)的列。这主要用于必须有一个列，而实际上并不使用值，用以与旧版本的应用程序相兼容。当需要只有两个值的列时CHAR(0)也很不错：定义了CHAR(0)NULL的列只占用一位，可只取值NULL和''（空字符串）

* [NATIONAL] VARCHAR(M) [CHARACTER SET charset_name] [COLLATE collation_name]

变长字符串。M表示最大列长度。M的范围是0到65535。VARCAHR的最大长度由最长行的大小（所有列的共享65535字节）和字符集决定。例如，utf8字符需要多达每个字符三个字节，所以VARCHAR列最多指定21844个字符。

MySQL存储VARCHAR时使用一个或两个字节的前缀+数据。长度前缀表示内容占用的字节数。当列长度不超过255个字节长度前缀占一个字节，当列长度超过255个字节长度前缀占两个字节。

VARCHAR是CHARACTER VARYING的简写。

### BINARY和VARBINARY类型

* BINARY(M)

BINARY类型与CHAR类型相似，不过存储的是二进制字节字符串而不是非二进制字符串。M表示列的长度（以字节为单位）

* VARBINARY(M)

VARBINARY类型与VARCHAR类型相似，不过存储的是二进制字节字符串而不是非二进制字符串。M表示最大列长度（以字节为单位）

### BLOB和TEXT类型

* TINYBLOB

这种BLOB列的最大长度是65535(2^8-1)个字节。存储每个TINYBLOB列时使用一个字节的前缀长度记录内容占用的字节数。

* BLOB[(M)]

BLOB列的最大长度是65535（2^16-1）个字节。存储每个BLOB列时使用两个字节的前缀长度记录内容占用的字节数。

* MEDIUMBLOB

这种BLOB列的最大长度是16777215（2^24-1）个字节。存储每个MEDIUMBLOB列时使用三个字节的前缀长度记录内容占用的字节数。

* LONGBLOB

这种BLOB列的最大长度是4294967295或者（2^32-1）个字节。LONGBLOB列的最大有效长度取决于客户端/服务器协议配置的最大包大小和可用内存。存储每个LONGBLOB列时使用四个字节的前缀长度记录内容占用的字节数。

* TINYTEXT [CHARACTER SET charset_name ] [COLLATE collation_name ]

这种TEXT列的最大长度是255（2^8-1）个字节。如果内容包含多字节字符，那么最大有效长度将减少。存储每个TINYTEXT时使用一个字节的前缀长度记录内容占用的字节数。

* TEXT[( M )] [CHARACTER SET charset_name ] [COLLATE collation_name ]

TEXT列的最大长度是65535（2^16-1）个字节。如果内容包含多字节字符，那么最大有效长度将减少。存储每个TEXT列时使用两个字节的前缀长度记录内容占用的字节。

可以给出该类型的可选长度M。如果指定了，那么MySQL会创建一个最小的，但可以容纳M字节长度的TEXT列。

* MEDIUMTEXT [CHARACTER SET charset_name ] [COLLATE collation_name ]

这种TEXT列的最大长度是16,777,215（224-1）个字节。如果内容包含多字节字符，那么最大有效长度将减少。存储每个MEDIUMTEXT列时使用三个字节的前缀长度记录内容占用的字节数。

* LONGTEXT [CHARACTER SET charset_name ] [COLLATE collation_name ]

这种TEXT列的最大长度是4,294,967,295或者4GB（232-1）个字节。如果内容包含多字节字符，那么最大有效长度将减少。LONGTEXT列的最大有效长度取决于客户端/服务器协议中配置的最大包大小和可用内存。存储每个LONGTEXT列时使用四个字节的前缀长度记录内容占用的字节数。

### ENUM类型

* ENUM( ' value1 ',' value2 ',...) [CHARACTER SET charset_name ] [COLLATE collation_name ]

枚举类型。属于字符串对象，允许从' value1 ',' value2 ',...,NULL或特殊的''错误值 列表中选取一个值。ENUM值内部用整数表示。

ENUM列最多可以有65,535个不同的元素（实际的限制小于3000个元素）。一张表可以不超过255个唯一的元素，列表定义中ENUM和SET列视为一组。

### SET类型

* SET(' value1 ',' value2 ',...) [CHARACTER SET charset_name ] [COLLATE collation_name ]

设置类型。属于字符串对象，可以有零个或多个值，必须从' value1 ',' value2 ',...中选取值，SET值内部用整数表示。

SET列最多可以有64个不同的元素。一张表可以不超过255个唯一的元素，列表定义中ENUM和SET列视为一组。

## 字符型数据存储格式

### 解析环境描述

#### 表结构定义

```
CREATE TABLE `string_table` (
  `col1` varchar(500) COLLATE utf8_bin DEFAULT NULL,
  `col2` char(60) COLLATE utf8_bin DEFAULT NULL,
  `col3` blob,
  `col4` set('a','b','c') COLLATE utf8_bin DEFAULT NULL,
  `col5` enum('one','two','three') COLLATE utf8_bin DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin
```

#### 表数据展示

```
root@localhost : gangshen 07:37:31> insert into string_table values('abcdefg','abc','abcdefghijklmnopqrstuvwxyz','c','two');
Query OK, 1 row affected (0.04 sec)

root@localhost : gangshen 07:40:27> select * from string_table;
+---------+------+----------------------------+------+------+
| col1    | col2 | col3                      | col4 | col5 |
+---------+------+----------------------------+------+------+
| abcdefg | abc  | abcdefghijklmnopqrstuvwxyz | c    | two  |
+---------+------+----------------------------+------+------+
1 row in set (0.00 sec)
```

然后从binlog文件中拿到，对应的Table_map_event和Writes_rows_event对应的字节内容，开始解析

### 元数据解析

#### Table_map_event字节解析

```
公共头部部分省略。。。
61 00 00 00 00 00 //table_id：小端存储，16进制转换成10进制为97
01 00 //flag
08 67 61 6e 67 73 68 65 6e 00 //database_name:1个字节是字符串长度，后接一个null-terminated string 第一个字节表示字符串长度为8，后面内容将16进制转换为ascii字符为gangshen
0c 73 74 72 69 6e 67 5f 74 61 62 6c 65 00//table_name: 1个字节是字符串长度，后接一个null-terminated string第一个字节表示字符串长度为5，后面内容将16进制转换为ascii字符为string_table
05 //columns count :packet integer类型，转换之后，数值为5 表示表中有5个字段
0f fe fc fe fe //column type :
09 //metadata length : packet integer类型,转换之后，数值为9，表示记录表中的metadata内容占用9个字节
dc 05 //col1
fe b4 //col2
02 //col3
f8 01 //col4
f7 01 //col5
1f //null_bits :int((column_count + 7) / 8)个字节 一个bit表示一个字段是否可以为null
22 6b 17 48 //checksum
```

从Table_map_event中可以按照上面的讲述，解析出表中所有的字段类型以及对应的元数据，按照顺序分别是：

第一个字段：0x0f=MYSQL_TYPE_VARCHAR 元数据0xdc05表示字段最多占用的字节数，因为是小端存储，所以该字段最多占用0x05dc=1500字节。这个占用的字节数取决于表的字符集，因为表的字符集为utf8，所以每个字符最多占用3个字节，所以该字段最多占用500*3个字节。

第二个字段：0xfe=MYSQL_TYPE_STRING，因为是MYSQL_TYPE_STRING类型，所以其真实的字段类型应该是元数据第一个字节（0xfe）和0x30做与运算之后的结果， 第二个字节表示该字段最多占用的字节数，所以该字段最多占用0xb4=180个字节

第三个字段：0x12=MYSQL_TYPE_BLOB 元数据0x02表示该字段类型为BLOB/TEXT类型

第四个字段：0x11=MYSQL_TYPE_STRING,所以其真实的字段类型应该是元数据的第一个字节（0xf8）和0x30做与运算之后的结果，即其真实类型是MYSQLTYPE_SET第二个字节表示该字段最多占用的字节数，所以该字段最多占用0x01个字节的内容。

第五个字段：0x11=MYSQL_TYPE_STRING，所以其真实的字段类型应该是元数据的第一个字节（0xf8）和0x30做与运算之后的结果，即其真实类型是MYSQLTYPE_SET第二个字节表示该字段最多占用的字节数，所以该字段最多占用0x01个字节的内容。


### 字符型数据“值”解析
#### Write_rows_log_event字节解析

```
公共头部部分省略。。。
61 00 00 00 00 00 //table_id: 小端存储，16进制转换为10进制为97
01 00 //flag:
02 00 //var header length :小端存储，16进制转换为10进制为2
05 //m_width :packet integer，表示表中字段数量
ff //before image: (m_width + 7) / 8字节
e0 //bitmap_bits :表中两个字段，插入的值都不为NULL
07 00 61 62 63 64 65 66 67 //col1
03 61 62 63 //col2
1a 00 61 62 63 64 65 66 67 68 69 6a 6b 6c 6d 6e 6f 70 71 72 73 74 75 76 77 78 79 7a //col3
04 //col4
02 //col5
46 1d 2a 69 //checksum
```


#### MYSQL_TYPE_VARCHAR字段解析

MYSQL_TYPE_VARCHAR类型字段由长度和字符串内容两部分组成，长度由由字段的元数据决定，字段的元数据表示该字段最大占用字节数。如果元数据小于255，则MYSQL_TYPE_VARCHAR类型使用一个字节表示字符串长度，长度后面紧接着就是字符串的内容；如果元数据大于等于255，则MYSQL_TYPE_VARCHAR类型使用两个字节表示字符串长度（小端存储），长度后面紧接这就是字符串的内容。

在上面的例子中，col1的字段类型为MYSQL_TYPE_VARCHAR类型，且元数据为0x05dc=2500大于255，所以使用两个字节表示字符串长度，所以字符串长度为0x0700，因为是小端存储，所以实际为0x0007，即字符串长度为7个字节。紧接着读取7个字节的内容0x61626364656667转换为ascii为abcdefg，所以col1的字段值为abcdefg

#### MYSQL_TYPE_STRING字段解析

MYSQL_TYPE_STRING类型字段由长度和字符串内容两部分组成，长度由由字段的元数据决定，字段的元数据表示该字段最大占用字节数。如果元数据小于255，则MYSQL_TYPE_STRING类型使用一个字节表示字符串长度，长度后面紧接着就是字符串的内容；如果元数据大于等于255，则MYSQL_TYPE_STRING类型使用两个字节表示字符串长度（小端存储），长度后面紧接这就是字符串的内容。

在上面的例子中，col2的字段类型是MYSQL_TYPE_STRING类型，且元数据中最大占用字节数为0xb4=180小于255，所以使用一个字节表示字符串长度，所以字符串长度为0x03=3个字节。紧接着读取3个字节的内容0x616263专户为ascii为abc，所以col2的字段值为abc

#### MYSQL_TYPE_BLOB字段解析

MYSQL_TYPE_BLOB类型字段由长度和字符串内容两部分组成，长度由字段的元数据决定，字段的元数据表示BLOB的类型。如果元数据是1，表示是TINYBLOB/TINYTEXT类型，则MYSQL_TYPE_BLOB类型使用一个字节表示字符串长度，长度后面紧接着就是字符串的内容；如果元数据是2，表示是BLOB/TEXT类型，则MYSQL_TYPE_BLOB类型使用两个字节表示字符串长度（小端存储），长度后面紧接着就是字符串内容；如果元数据是3，表示是MEDIUMBLOB/MEDIUMTEXT类型，则MYSQL_TYPE_BLOB类型使用三个字节表示字符串长度（小端存储），长度后面紧接着就是字符串内容；如果元数据是4，表示是LONGBLOB/LONGTEXT类型，则MYSQL_TYPE_BLOB类型使用四个字节表示字符串长度（小端存储），长度后面紧接着就是字符串内容。

在上面的例子中，col3字段的类型是MYSQL_TYPE_BLOB类型，且元数据为0x02，表示类型为BLOB/TEXT类型，说明该字段使用两个字节表示字符串的长度。所以字符串长度为0x1a00，因为是小端存储，所以字符串长度为0x001a=26，紧接着读取26个字节的内容0x6162636465666768696a6b6c6d6e6f707172737475767778797a转换为ascii为abcdefghijklmnopqrstuvwxyz，所以col3的字段值为abcdefghijklmnopqrstuvwxyz。

#### MYSQL_TYPE_SET字段解析

MYSQL_TYPE_SET类型字段内容取决于字段定义SET的数量，表示字段值所需字节数量可以从字段元数据的第二个字节获取到，第二个字节表示内容长度。MYSQL_TYPE_SET类型字段内容采用压缩的表示方法，对应比特位置1则表示值是该位，但是从binlog中无法解析出SET类型内容的具体类型，只能解析出对应的比特位表示方式。

在上面的例子中，col4字段的类型是MYSQL_TYPE_SET，且元数据中表示该字段使用0x01个字节表示字段值，所以col4在Write_rows_log_event占用1个字节的内容，为0x04，转换为二进制为b'100'，从右往左第3位比特位上是1，表示值为set定义中第三个。从Write_rows_log_event中，无法解析出set定义的每个项，查看建表语句，得知set定义中第三个为c，即col4字段的值为c

#### MYSQL_TYPE_ENUM字段解析

MYSQL_TYPE_ENUM类型字段内容取决于字段定义ENUM数量，表示字段值所需字节数量可以从字段元数据的第二个字节获取到，第二个字节表示内容长度。

当元数据第二个字节等于1的时候，MYSQL_TYPE_ENUM字段值使用一个字节表示，当元数据第二个字节等于2的时候，MYSQL_TYPE_ENUM字段值使用两个字节表示。

在上面的例子中，col5字段的类型是MYSQL_TYPE_ENUM，且元数据中表示该字段使用0x01个字节表示字段值，所以col5在Write_rows_log_event中占用1个字节的内容，为0x02，转换为10进制为2，即表示值为enum定义中的第2个。在Write_rows_log_event中，无法解析出enum定义的每个项，查看建表语句，得知enum定义中第二个值为two，即col5字段的值为two。


