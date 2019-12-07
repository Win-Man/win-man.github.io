---
title: MySQL Binlog(八)——MySQL字段存储格式——数值类型字段类型
date: 2019-12-07 16:45:13
tags:
- MySQL
- Binlog
categories:
- MySQL
---

# MySQL Binlog(八)——MySQL字段存储格式——数值类型字段类型

## 前言

从本文开始，我会开始介绍MySQL中常见类型字段具体的存储格式，了解这部分内容之后，我们可以用于解析前面讲到的Write_rows_log_event、Update_rows_event、Delete_rows_event中具体关于数据的部分了，可以解析出某个字段具体记录了哪些值，具体的值是什么。

我们首先从数值型字段类型开始，数值型字段类型又可以分为整数类型、定点数数据类型、浮点数数据类型、比特数据类型。

## 数值型字段类型介绍

### 整数类型 

* TINYINT[(M)] [UNSIGNED] [ZEROFILL]

整数，用一个字节表示。分为有符号类型和无符号类型，有符号类型的TINYINT范围是-128到127，无符号类型的TINYINT范围是0到255。

* SMALLINT[(M)] [UNSIGNED] [ZEROFILL]

整数，用两个字节表示。分为有符号类型和无符号类型，有符号类型的SMALLINT范围是-32768到32767，无符号类型的SMALLINT范围是0到65535

* MEDIUMINT[(M)] [UNSIGNED] [ZEROFILL]

整数，用三个字节表示。分为有符号类型和无符号类型，有符号类型的MEDIUMINT范围是-8388608到8388607，无符号类型的MEDIUMINT范围是0到16777215

* INT[M] [UNSIGNED] [ZEROFILL]

整数，用四个字节表示。分为有符号类型和无符号类型，有符号类型的INT范围是-2147483648到2147483647，无符号类型的INT范围是0到4294967295

* BIGINT[M] [UNSIGNED] [ZEROFILL]

整数，用八个字节表示。分为有符号类型和无符号类型，有符号类型的BIGINT范围是-9223372036854775808到9223372036854775807，无符号类型的BIGINT范围是0到18446744073709551615

`SERIAL`是`BIGINT UNSIGNED NOT NULL AUTO_INCREMENT UNIQUE`的别名

* BOOL,BOOLEAN

TINYINT(1)的别名。

* INTEGER[(M)] [UNSIGNED] [ZEROFILL]

INT的别名

### 定点数数据类型 DECIMAL,NUMERIC

* DECIMAL[(M|,D)] [UNSIGNED] [ZEROFILL]

小数，M表示数据的精度，表示数据的总长度，小数点和符号不占长度；D表示数据的标度，指小数点后面数字的位数。

M的范围是0到65，默认为10；D的范围是0到30，默认为0。

* DEC[(M[,D])] [UNSIGNED] [ZEROFILL], NUMERIC[(M[,D])] [UNSIGNED] [ZEROFILL], FIXED[(M[,D])] [UNSIGNED] [ZEROFILL]

DECIMAL的别名

### 浮点数数据类型 FLOAT,DOUBLE

* FLOAT[M,D] [UNSIGNED] [ZEROFILL]

单精度浮点数，用四个字节表示。允许的值是-3.402823466E+38 到-1.175494351E-38, 0,  1.175494351E-38 到 3.402823466E+38。但是这只是理论的数值范围，实际的范围区间取决于硬件平台。

M表示数据的精度，表示数据的总长度，小数点和符号不占长度；D表示数据的标度，指小数点后面数字的位数。M和D的默认值取决于硬件平台。单精度的浮点数大约是7个小数点。

* DOUBLE[(M,D)] [UNSIGNED] [ZEROFILL]

双精度浮点数，用八个字节表示。允许的值是-1.7976931348623157E+308 到 -2.2250738585072014E-308, 0, 和 2.2250738585072014E-308 到 1.7976931348623157E+308

。但是这只是理论的数值范围，实际的范围区间取决于硬件平台。

M表示数据的精度，表示数据的总长度，小数点和符号不占长度；D表示数据的标度，指小数点后面数字的位数。M和D的默认值取决于硬件平台。双精度的浮点数大约是15个小数点。

* DOUBLE PRECISION[(M,D)] [UNSIGNED] [ZEROFILL], REAL[(M,D)] [UNSIGNED] [ZEROFILL

DOUBLE的别名

### 比特数据类型 BIT

* BIT[(M)]

比特数据类型。M表示每个值占用的bit位数，M的范围是1到64。如果M没有指定，则默认值为1。

## 数值型数据解析

### 解析环境描述

#### 表结构定义

```
CREATE TABLE `number_table` (
  `col1` tinyint(4) DEFAULT NULL,
  `col2` smallint(6) DEFAULT NULL,
  `col3` mediumint(9) DEFAULT NULL,
  `col4` int(11) DEFAULT NULL,
  `col5` bigint(20) DEFAULT NULL,
  `col6` decimal(25,10) DEFAULT NULL,
  `col7` float DEFAULT NULL,
  `col8` double DEFAULT NULL,
  `col9` bit(5) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin
```

#### 表数据展示

```
root@localhost : gangshen 02:59:24> insert into number_table values(2,-22,222,-2222,22222,123123123123.112233,123.1,123.2,b'00110');
Query OK, 1 row affected (0.04 sec)

root@localhost : gangshen 03:03:39> select * from number_table;
+------+------+------+-------+-------+-------------------------+-------+-------+------+
| col1 | col2 | col3 | col4  | col5  | col6                    | col7  | col8  | col9 |
+------+------+------+-------+-------+-------------------------+-------+-------+------+
|    2 |  -22 |  222 | -2222 | 22222 | 123123123123.1122330000 | 123.1 | 123.2 |    |
+------+------+------+-------+-------+-------------------------+-------+-------+------+
1 row in set (0.00 sec)
```

然后从binlog文件中拿到对应的Table_map_event和Writes_rows_event对应的字节内容，开始解析

### 元数据存储格式

#### Table_map_event字节数据解析

```
公共头部部分省略
5d 00 00 00 00 00 //table_id
01 00 //flag
08 67 61 6e 67 73 68 65 6e 00 //database_name :1个字节是字符串长度，后接一个null-terminated string 第一个字节表示字符串长度为8，后面内容将16进制转换为ascii字符为gangshen
0c 6e 75 6d 62 65 72 5f 74 61 62 6c 65 00 //table_name: 1个字节是字符串长度，后接一个null-terminated string第一个字节表示字符串长度为12，后面内容将16进制转换为ascii字符为number_table
09 //columns count :packet integer类型，转换之后，数值为9 表示表中有9个字段
01 02 09 03 08 f6 04 05 10 //column types
06 //metadata_length
19 0a //col6 metadata
04 //col7 metadata
08 //col8 metadata
05 00 //col9 metadata
ff 01 //null_bits :int((column_count + 7) / 8)个字节 一个bit表示一个字段是否可以为null
e2 06 a7 b1 //checksum
```

从Table_map_event中可以参照上节描述，解析出表中所有的字段类型以及对应的元数据，按照顺序分别是：

第一个字段：0x01=MYSQL_TYPE_TINY 无元数据

第二个字段：0x02=MYSQL_TYPE_SHORT 无元数据

第三个字段：0x09=MYSQL_TYPE_INT24 无元数据

第四个字段：0x03=MYSQL_TYPE_LONG 无元数据

第五个字段：0x08=MYSQL_TYPE_LONGLONG 无元数据

第六个字段：0xf6=MYSQL_TYPE_NEWDECIMAL 元数据表示精度为25(0x19)，标度为10(0x0a)

第七个字段：0x04=MYSQL_TYPE_FLOAT 元数据表示该字段需要使用4(0x04)个字节表示

第八个字段：0x05=MYSQL_TYPE_DOUBLE 元数据表示该字段需要使用8(0x08)个字节表示

第九个字段：0x10=MYSQL_TYPE_BIT 元数据表示该字段中一共有5(0x0500)个比特位

### 数值型数据“值”解析

#### Write_rows_log_event 字节数据解析

```
公共头部部分省略。。。
5d 00 00 00 00 00 //table_id:
01 00 //flag
02 00 //var header length
09 //m_width :packet integer，表示表中字段数量
ff ff //before image: (m_width + 7) / 8字节
00 fe //bitmap_bits :表中两个字段，插入的值都不为NULL
02 //col1
ea ff //col2
de 00 00 //col3
52 f7 ff ff //col4
ce 56 00 00 00 00 00 00 //col5
80 00 7b 07 56 b5 b3 06 b0 8a 28 00 //col6
33 33 f6 42 //col7
cd cc cc cc cc cc 5e 40 //col8
06 //col9
5e 6d 11 fa //checksum
```

接下来，对应write_row_event中各个字段的二进制，我们按照顺序解析各个字段的值

#### MYSQL_TYPE_TINY字段解析

MYSQL_TYPE_TINY需要使用1个字节表示，小端存储。
上面的例子中，col1的类型是MYSQL_TYPE_TINY，在所以在Write_rows_log_event中字段值的内容占用1个字节，为0x02，转换成10进制就是2

####  MYSQL_TYPE_SHORT字段解析

MYSQL_TYPE_SHORT需要使用2个字节表示，小端存储。

上面的例子中，col2的类型是MYSQL_TYPE_SHORT，所以在Write_rows_log_event中字段值的内容占用2个字节，为0xEAFF，因为是小端存储，所以实际的顺序为0xFFEA，数据采用补码的方式表示有符号数，所以转换成10进制数值为-22

#### MYSQL_TYPE_INT24字段解析

MYSQL_TYPE_INT24需要使用3个字节表示，小端存储。

上面的例子中，col3的类型是MYSQL_TYPE_INT24，所以在Write_rows_log_event中字段值的内容占用3个字节，为0xDE0000，因为是小端存储，所以实际的顺序为0x0000DE，数据采用补码的方式表示有符号数，所以转换成10进制数值为222

#### MYSQL_TYPE_LONG字段解析

MYSQL_TYPE_LONG需要使用4个字节表示，小端存储。

上面的例子中，col4的类型是MYSQL_TYPE_LONG，所以在Write_rows_log_event中字段值的内容占用4个字节，为0x52F7FFFF，因为是小端存储，所以实际的顺序为0xFFFFF752，数据采用补码的方式表示有符号数，所以转换成10进制数值为-2222

#### MYSQL_TYPE_LONGLONG 字段解析

MYSQL_TYPE_LONGLONG需要使用8个字节表示，小端存储。

上面的例子中，col5的类型是MYSQL_TYPE_LONGLONG，所以在Write_rows_log_event中字段值的内容占用8个字节，为0xCE56000000000000，因为是小端存储，所以实际的顺序为0x00000000000056CE，数据采用补码的方式表示有符号数，所以转换成10进制数值为22222

#### MYSQL_TYPE_DECIMAL字段解析

MYSQL_TYPE_DECIMAL采用压缩存储的方式，将9个数字（十进制）使用4个字节来表示；每9位数字使用4个字节表示，剩余未满9位数字的，所需字节数参照下表。并且DECIMAL中整数部分与小数部分是单独分开表示的。

| 剩余位数 | 所需字节数 |
| ---------- | ------------- |
| 0 | 0 |
| 1 | 1 |
| 2 | 1 |
| 3 | 2 |
| 4 | 2 |
| 5 | 3 |
| 6 | 3 |
| 7 | 4 |
| 8 | 4 |

MYSQL_TYPE_DECIMAL类型的字段，在存储数值的时候，首先将数值按照整数部分以及小数部分划分。

整数部分从低位到高位，每9位数字使用4个字节存储，将每9位数字与正负号（正数：0，负数：-1）做异或运算之后，以大端、有符号整数存储，对于剩余不足9位数字的部分，参照表格存储，将剩余数字与正负号（正数：0，负数：-1）做异或运算之后并异或上一个0x80，以大端、有符号整数存储。

小数部分从高位到低位，每9位数字使用4个字节存储，将每9位数字与正负号（正数：0，负数：-1）做异或运算之后，以大端、有符号整数存储，对于剩余不足9位数字的部分，参照表格存储，将剩余数字与正负号（正数：0，负数：-1）做异或运算之后并异或上一个0x80，以大端、有符号整数存储。

例如数字12.34，整数部分12，2位数字需要一个字节，0x0C与0做异或操作结果是0x0C，小数部分34，2位数字需要一个字节，0x22与0做异或操作结果是0x22，所以数字12.34的MYSQL_TYPE_DECIMAL在binlog中以0xOC22方式存储。

上面的例子中，col6的类型是MYSQL_TYPE_NEWDECIMAL，按照MYSQL_TYPE_NEWDECIMAL存储要求，需要（(15/9)*4+3 + (10/9)*4+1）=12个字节，所以在Write_rows_log_event中字段值的内容占用12个字节，为0x80007b0756b5b306b08a2800，其中整数部分使用7个字节存储，内容为0x80007b0756b5b3，小数部分使用5个字节存储，内容为0x06b08a2800。

* 首先解析整数部分

MYSQL_TYPE_NEWDECIMAL类型值的符号可以从第一个字节中获取到，将第一个字节和0x80做与操作，如果结果不为0表示该值为正数，记mask=0；如果结果为0表示该值为负数，记mask=-1。

因为MYSQL_TYPE_NEWDECIMAL类型使用第一个字节的最高位来表示值的正负，所以在解析具体的值的时候，需要将该符号位忽略，所以真实的数值内容需要将第一个字节与0x80做异或运算。即上述内容中，整数部分数值内容为0x00007b0756b5b3，小数部分数值内容为0x06b08a2800。

整数部分因为最多存储15位数据，所以使用4+6=7个字节存储。因为整数部分是从低位到高位（从右往左）每9位数字使用4个字节存储，剩余位数按照要求存储，所以这7个字节中，前3个字节表示前（15-9）=6位，剩余4个字节表示后9位。将前三个字节0x00007b转换成10进制为123，因为存储的时候将数值与正负号（正数：0，负数：-1）做过异或运算，所以解析的时候需要将解析出来的123与mask做异或运算，即实际前6位数值为123。后4个字节0x0756b5b3转换成10进制为123123123，因为存储的时候将数值与正负号（正数：0，负数：-1）做过异或运算，所以解析的时候需要将解析出来的123与mask做异或运算，即后9位数字为123123123。将整数部分连接起来就是123123123123。

* 接着解析小数部分

小数部分因为最多存储10位数据，所以使用4+1=5个字节存储。因为小数部分是从高位到低位（从左往右）每9位数字使用4个字节存储，剩余位数按照要求存储，所以这5个字节中，前4个字节表示9位数字，剩余1个字节表示（10-9）=1位。前4个字节0x0ab08a28转换成10进制是112233000，因为存储的时候将数值与正负号（正数：0，负数：-1）做异或运算，所以解析的时候需要将解析出来的112233000与mask做异或运算，即实际前9位数字为112233000。后一个字节0x00转换成10进制为0，因为存储的时候将数值与正负号（正数：0，负数：-1）做异或运算，所以解析的时候需要将解析出来的0与mask做异或运算，即实际后1位数字为0。将小数部分连接起来就是1122330000。

将解析出来的整数部分与小数部分连接在一起就是123123123123.1122330000

#### MYSQL_TYPE_FLOAT字段解析

MYSQL_TYPE_FLOAT类型需要四个字节表示。

上面的例子中，col7的类型是MYSQL_TYPE_FLOAT，所以在Write_rows_log_event中字段值的内容占用4个字节，为0x3333f642，将字节内容强转成float类型的数据(MySQL在解析binlog的时候，也是将字节内容强转成float类型)，得到的结果取决于不同的编程语言。比如在Python中，0x3333f642内容强转成float的结果就是123.099998。

#### MYSQL_TYPE_DOUBLE字段解析

MYSQL_TYPE_DOUBEL类型需要八个字节表示

上面的例子中，col8的类型是MYSQL_TYPE_DOUBLE，所以在Write_rows_log_event中字段值的内容占用8个字节，为0xcdcccccccccc5e40，将字节内容强转成double类型的数据（MySQL在解析binlog的时候，也是将字节内容强转成double类型），得到的结果取决于不同的编程语言。比如在Python中，0xcdcccccccccc5e40内容强转成double类型就是123.200000。

#### MYSQL_TYPE_BIT字段解析

MYSQL_TYPE_BIT类型需要（M+7）/8，M是字段定义的比特位数。

上面的例子中，col9类型是MYSQL_TYPE_BIT，对应的元数据中定义该字段为5个比特位，所以需要（5+7）/8=1个字节存储数值。所以在Write_row_event中字段值的内容占用1个字节，为0x06，对应的二进制表示方式为b'00110'，所以该字段的值为b'00110'。

