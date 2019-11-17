---
title: MySQL Binlog(二)——MySQL 协议基础
date: 2019-11-17 10:41:13
tags:
- MySQL
- Binlog
categories:
- MySQL
---
# MySQL Binlog(二)——MySQL 协议基础

## 1. 前言

上篇讲了MySQL的复制原理，这篇文章来讲讲建立复制过程中使用的MySQL通讯协议。MySQL协议是用于MySQL中Server端与Client端进行通讯的协议，MySQL主备复制也遵守Server和Client端通讯协议。

## 2. 基础数据类型

MySQL协议传输中的数据包含两种类型，一种是数值型数据，另一种是字符型数据。

### 2.1 数值型数据

* Protocol::FixedLengthInteger 

MySQL使用Protocol::FixedLenthInteger 表示固定长度的整数

| 类型 | 长度 | 对应函数 | 实际对应类型 | 
| ----- | ----- | ---------- | ---------------- |
| int<1> | 1 byte | | |
| int<2> | 2 byte | int2store() | uint16 |
| int<3> | 3 byte | int3store() | uint |
| int<4> | 4 byte | int4store() | uint32 |
| int<6> | 6 byte | int6store() | ulonglong |
| int<8> | 8 byte | int8store() | ulonglong |



* Protocol::LengthEncodedInteger 

使用Protocol::LengthEncodedInteger类型表示的整数，会根据表示的整数大小占用不同的字节数（1，3，4或9个字节）

| 整数范围 | 表示方式 |示例|
| ----- | ----------- |---|
| 0≤N＜251 | 1 byte |0x08 表示整数8|
| 251≤N＜2^16 | 0xFC + 2 byte |0xFC0008 = 0xFC + 0x0008 表示整数260 |
| 2^16≤N＜2^24 | 0xFD + 3 byte | 0xFD123456 = 0xFD + 0x123456  表示整数1193299 |
| 2^24≤N＜2^64 | 0xFE + 8 byte | 0xFE0000000000000009 = 0xFE + 0x0000000000000009 表示整数263 |


### 2.2 字符型数据

* Protocol::FixedLengthString 固定长度的字符串

字符串用固定长度表示，例如ERR_Packet中的 `sql_state` 部分就是固定5个字节长度

* Protocol::NullTerminatedString
* 
以0x00结尾的字符串

* Protocol::VariableLengthString

在程序运行过程中动态确定长度的字符串

* Protocol::LengthEncodedString

在字符串开始用Protocol::LengthEncodedInteger类型整数表示字符串的长度

* Protocol::RestOfPacketString

字符串是整个数据包的最后一个组成部分，字符串的长度等于整个数据包的长度减去当前位置

## 3. 基本协议数据包格式

MySQL中server端与client端进行数据交换的时候，会将要交换的数据，按照上述的协议数据类型封装成数据包，然后通过MySQL协议进行数据包的传输。


### 3.1 基础数据包的格式定义

相关代码位置：

```
sql/net_serv.cc -> net_write_command()函数
```

|名字 | 基本数据类型 | 类型 | 描述 |
| ---- | --------------- | -----  | ------ |
| payload_length | Protocol::FixedLengthInteger |  int<3> | 数据包中有效信息的长度，不包括数据包头部4个字节的长度 |
| sequence_id | Protocol::FixedLengthInteger | int<1> | 数据包序列号 |
| payload | Protocol::VariableLengthString | string<var>  | 数据包的有效信息 |


### 3.2 基础数据包的格式解析

MySQL协议中数据包固定由 `payload_length` + `sequence_id` + `payload` 组成。每个数据包前3个字节固定存储`payload_length`信息，即数据包中有效信息`payload`的长度，第4个字节表示该数据包在此次通信中的序号。因为`payload_length`为int<3>类型，所以能表示的最大`payload`部分长度为2^24-1个字节，也就是说一个数据包能传输的数据量最大为2^24-1字节（合16MB）。

那么对于一次请求中，数据量超过16MB的数据，MySQL是如何处理的呢？

对于`payload`部分大于等于16MB的，MySQL会将数据进行切分，每16MB为一部分，然后将切分的数据按照数据包的格式封装，进行发送，直到切分的数据部分长度小于16MB。

## 4. 常见数据包示例

上面介绍了MySQL协议中数据包的基础格式，这一节介绍MySQL协议中常见的数据包类型，不同类型的数据包，其格式都是符合上述的基础数据包格式定义的。但是对于不同数据包类型，其`payload`部分的数据格式是不同的。

### 4.1 OK_Packet

对于Client发送的请求，如果执行成功，则Server端会向Client端发送OK_Packet告诉Client端请求执行成功。但是在5.7.5版本之后，OK Packet也可以用于表示EOF信息，原先的 EOF_Packet被废弃了，EOF Packet后面章节会详细描述。

#### 4.1.1 OK_Packet的格式定义

相关代码位置：

```
sql/protocol_classic.cc -> net_send_ok()函数
```

| 名字 | 基本数据类型 | 类型  | 描述 |
| ---- | --------------- | ----- | ------ |
| header | Protocol::FixedLengthInteger | int<1> | 0x00表示该数据包是OK_Packet，0xFE表示该数据包是ERR_Packet |
| affect_rows | Protocol::LengthEncodedInteger  | int<lenenc>  | 请求在Server端执行影响的记录行数 |
| last_insert_id | Protocol::LengthEncodedInteger  | int<lenenc> | 对于INSERT请求，表示最近插入的那行记录的自增字段值 |
| status_flags| Protocol::FixedLengthInteger | int<2>  | Server端的状态信息，如果Client发送的请求中带有CLIENT_PROTOCOL_41标志,会有这一部分，否则为空 |
| warnings| Protocol::FixedLengthInteger | int<2> | 警告的数量，如果Client发送的请求中带有CLIENT_PROTOCOL_41标志,会有这一部分，否则为空 |
| status_flag | Protocol::FixedLengthInteger | int<2> | 如果Client发送的请求中带有CLIENT_TRANSACTIONS标志,会有这一部分，否则为空 |
| messages | Protocol::LengthEncodedString | string<lenenc> | 当前会话状态改变的信息，如果Client发送的请求中带有CLIENT_SESSION_TRACK标志，会有这一部分，否则为空 |
| session_state_change_info | Protocol::LengthEncodedString | string<lenenc> | 当前会话状态改变的信息，如果Client发送的请求中带有SERVER_SESSION_STATE_CHANGED标志,会有这一部分，否则为空 |
| messages | Protocol::RestOfPacketString | string<EOF> | 请求执行结果信息，如果不包含信息，该部分为空 |
其中`int<lenenc>`和`string<lenenc>`中的lenenc表示的是LengthEcode。

`status_flag`字段中记录的是Server端的状态信息，MySQL支持的`server_status`类型：

```
#define SERVER_STATUS_IN_TRANS 1
#define SERVER_STATUS_AUTOCOMMIT 2    /* Server in auto_commit mode */
#define SERVER_MORE_RESULTS_EXISTS 8 /* Multi query - next query exists */
#define SERVER_QUERY_NO_GOOD_INDEX_USED 16
#define SERVER_QUERY_NO_INDEX_USED 32
#define SERVER_STATUS_CURSOR_EXISTS 64
#define SERVER_STATUS_LAST_ROW_SENT 128
#define SERVER_STATUS_DB_DROPPED 256 /* A database was dropped */
#define SERVER_STATUS_NO_BACKSLASH_ESCAPES 512
#define SERVER_STATUS_METADATA_CHANGED 1024
#define SERVER_QUERY_WAS_SLOW 2048
#define SERVER_PS_OUT_PARAMS 4096
#define SERVER_STATUS_IN_TRANS_READONLY 8192
#define SERVER_SESSION_STATE_CHANGED (1UL << 14)

```

当表示两个及两个以上状态时，将代表多个状态的值做`^`运算即可。

#### 4.1.2 OK Packet的格式解析
上面讲到5.7.5版本之后，OK Packet既可以表示OK Packet也可以表示EOF Packet，那么在5.7.5之后如何区分是OK Packet 还是 EOF Packet呢？
* OK_Packet:头部部分是0x00，并且数据包的长度是大于7
* EOF_Packet:头部部分是0xFE，并且数据包的长度是小于9

为了保证5.7.5之前版本与之后版本的兼容性，5.7.5之后的MySQL 版本中的client端都会发送`CLIENT_DEPRECATE_EOF`标志。旧版本client不会发送该标志，所以Server端还是采用专门的EOF_Packet（下面会具体介绍）表示EOF信息，新版本的client在向Server端发送请求时会带上该标志，但是对于旧版本的Server端，无法识别该标志，所以依然是采用专门的EOF_Packet表示EOF信息。

client端发送带`CLIENT_PROTOCOL_41`标志的数据包示例（内容为16进制格式）：

```
07 00 00 #数据包长度
02           #数据包序号
00           #头部
00          #影响的记录行数
00          #上一次插入的记录自增值
02 00     #状态标志
00 00     #警告数量
```

各分段数据含义：

* 数据包长度：MySQL协议中固定格式，示例中`07 00 00`表示0x000007，代表数据包长度为7个字节
* 数据包序号：MySQL协议中固定格式，示例中`00`表示数据包序号为2
* 头部：示例中`ff`表示该数据包为OK_Packet
* 影响的记录行数：示例中`00`表示0x00，代表本次client请求共影响0行记录
* 上一次插入的记录自增值：实例中`00`表示0x00
* 状态标志：因为是小端存储，所以示例中`02 00`表示0x0002，代表Server端开启了auto_commit
* 警告的数量：因为是小端存储，所以示例中`00 00`表示0x0000，表示告警数量为0

#### 4.1.3 Session State Information

OK_Packet中也可以记录会话级别的状态改变信息，包括系统变量信息、当前会话使用的schema信息、会话级别状态改变信息、事务特性、事务状态信息。当Client端发送的请求中带有`SERVER_SESSION_STATE_CHANGED`标志，则在Server端返回的OK_Packet中会带有这些信息。OK_Packet一共支持如下几种会话级别状态信息：

```
enum enum_session_state_type
{
  SESSION_TRACK_SYSTEM_VARIABLES, /* Session system variables */
  SESSION_TRACK_SCHEMA, /* Current schema */
  SESSION_TRACK_STATE_CHANGE, /* track session state changes */
  SESSION_TRACK_GTIDS,
  SESSION_TRACK_TRANSACTION_CHARACTERISTICS, /* Transaction chistics */
  SESSION_TRACK_TRANSACTION_STATE /* Transaction state */
};
```

具体可参考官方文档：https://dev.mysql.com/doc/dev/mysql-server/latest/page_protocol_basic_ok_packet.html



###  4.2 ERR_Packet
该数据包用于返回出现错误信息。
相关代码位置：

```
sql/protocol_classic.cc -> net_send_error_packet()函数
```

#### 4.2.1 ERR_Packet的格式定义
在5.7.5之前版本中，ERR_Packet采用单独表示格式，5.7.5之后，ERR_Packet与OK_Packet复用相同格式。以下是针对5.7.5之前版本介绍ERR_Packet格式定义。

|名字 | 基本数据类型 | 类型 | 描述 |
| ---- | --------------- | ----- | ------ |
| header | Protocol::FixedLengthInteger | int<1> | 用0xFF表示该数据包是一个ERR Packet |
| error_no | Protocol::FixedLengthInteger | int<2> | 表示错误的类型，小端存储|
| sql_state_marker | Protocol::VariableLengthString | string[1] | 如果Client发送的请求中带有CLIENT_PROTOCOL_41标志，会有这一部分，否则为空 |
| sql_state | Protocol::VariableLengthString | string[5] | 如果Client发送的请求中带有CLIENT_PROTOCOL_41标志，会有这一部分，否则为空 |
| err_message  | Protocol::RestOfPacketString | string<EOF> | 可读的错误信息详情 |

 

#### 4.2.2 ERR_Packet的格式解析

数据包示例（内容为16进制格式）：

```
4b 00 00  #数据包长度
02            #数据包序号
ff              #ERR_Packet标识
15 04       #错误号
23 32 38 30 30 30 41 63 63 65 73 73 20 64 65 6e 69 65 64 20 66 6f 72 20 75 73 65 72 20 27 72 65 70 6c 27 40 27 31 32 31 2e 31 32 31 2e 30 2e 36 34 27 20 28 75 73 69 6e 67 20 70 61 73 73 77 6f 72 64 3a 20 59 45 53 29  #错误信息
```

各分段数据含义：

* 数据包长度：MySQL协议中固定格式，示例中`4b 00 00`表示0x00004b，代表数据包长度为75个字节
* 数据包序号：MySQL协议中固定格式，示例中`00`表示数据包序号为0
* 头部：示例中`ff`表示该数据包为ERR_Packet
* 错误号：因为是小端存储，所以示例中`15 04`对应16进制0x0415，转换为10进制为1045
* 错误信息：示例中的内容可以转换为`Access denied for user 'repl'@'121.121.0.64'(usingpassword:YES)`

### 4.3 EOF_Packet

表示当前会话可以结束了，结束连接的信号。
相关代码位置:

```
sql/protocol_classic.cc -> write_eof_packet()函数
```

#### 4.3.1 EOF_Packet的格式定义

|名字 | 基本数据类型 | 类型 | 描述 |
| ---- | --------------- | ----- | ------ |
| header | Protocol::FixedLengthInteger  | int<1> | 用0xFE表示该数据包是一个ERR_Packet |
| warnings | Protocol::FixedLengthInteger  | int<2> | 警告的数量，如果Client发送的请求中带有CLIENT_PROTOCOL_41标志，会有这一部分，否则为空，小端存储 |
| server_status | Protocol::FixedLengthInteger  | int<2> | Server端的状态信息，如果Client发送的请求中带有CLIENT_PROTOCOL_41标志,会有这一部分，否则为空，小端存储 |

其中`server_status`内容参照OK_Packet中`status_flags`字段。

#### 4.3.2 EOF_Packet的格式解析

数据包示例（内容为16进制格式）：

```
05 00 00  #数据包长度
05            #数据包序号
fe             #头部
00 00       #警告的数量
02 00       #Server端的状态信息
```

各分段数据含义：

* 数据包长度：MySQL协议中固定格式，示例中`05 00 00`表示0x000005，代表数据包长度为5个字节
* 数据包序号：MySQL协议中固定格式，示例中`05`表示数据包序号为5
* 头部：标识是否是EOF数据包，0xFE标识该数据包是EOF_Packet
* 警告的数量：因为是小端存储，所以示例中`00 00`表示0x0000，表示告警数量为0
* Server端的状态信息：因为是小端存储，所以示例中`02 00`表示0x0002，代表Server端开启了auto_commit

### 4.4 Query_Packet

#### 4.4.1 Query_Packet的格式定义

|名字 | 基本数据类型 | 类型 | 描述 |
| ---- | --------------- | ----- | ------ |
| payload_length | Protocol::FixedLengthInteger  | int<3> | 数据包中有效信息的长度，不包括数据包头部4个字节的长度，小端存储 |
| sequence_id | Protocol::FixedLengthInteger  | int<1> | 数据包序列号 |
| command_type | Protocol::FixedLengthInteger  | int<1> | 要发送的请求的类型 |
| command | Protocol::RestOfPacketString  | string<EOF> | 请求内容 |

#### 4.4.2 Query_Packet的格式解析

以`SET @master_binlog_checksum`语句请求为例

数据包示例（内容为16进制格式）：

```
36 00 00    #数据包长度
00              #数据包序号
03              #发送的command类型 0x03表示的是COM_QUERY类型的命令
53 45 54 20 40 6d 61 73 74 65 72 5f 62 69 6e 6c 6f 67 5f 63 68 65 63 6b 73 75 6d 3d 20 40 40 67 6c 6f 62 61 6c 2e 62 69 6e 6c 6f 67 5f 63 68 65 63 6b 73 75 6d   #要发送的请求语句 //SET @master_binlog_checksum=@@global.binlog_checksum
```

各分段数据含义： 

* 数据包长度：MySQL协议中固定格式，示例中`36 00 00`表示0x000036，代表数据包长度为36个字节
* 数据包序号：MySQL协议中固定格式，示例中`00`表示数据包序号为0
* COMMAND类型：表示发送的请求的类型，MySQL支持的命令类型如下：

```
enum enum_server_command
{
  COM_SLEEP,COM_QUIT,
  COM_INIT_DB,
  COM_QUERY,
  COM_FIELD_LIST,
  COM_CREATE_DB,
  COM_DROP_DB,
  COM_REFRESH,
  COM_SHUTDOWN,
  COM_STATISTICS,
  COM_PROCESS_INFO,
  COM_CONNECT,
  COM_PROCESS_KILL,
  COM_DEBUG,
  COM_PING,
  COM_TIME,
  COM_DELAYED_INSERT,
  COM_CHANGE_USER,
  COM_BINLOG_DUMP,
  COM_TABLE_DUMP,
  COM_CONNECT_OUT,
  COM_REGISTER_SLAVE,
  COM_STMT_PREPARE,
  COM_STMT_EXECUTE,
  COM_STMT_SEND_LONG_DATA,
  COM_STMT_CLOSE,
  COM_STMT_RESET,
  COM_SET_OPTION,
  COM_STMT_FETCH,
  COM_DAEMON,
  COM_BINLOG_DUMP_GTID,
  COM_RESET_CONNECTION,
  COM_END
};
```

* 要发送的请求语句：将要发送的语句转换为ASCII对应的16进制，示例中的内容可以转换为：`SET @master_binlog_checksum=@@global.binlog_checksum`

## 5. 参考链接

* https://dev.mysql.com/doc/dev/mysql-server/latest/page_protocol_basic_dt_integers.html
* https://dev.mysql.com/doc/dev/mysql-server/latest/page_protocol_basic_dt_strings.html
* https://dev.mysql.com/doc/dev/mysql-server/latest/page_protocol_basic_packets.html
* https://dev.mysql.com/doc/dev/mysql-server/latest/page_protocol_basic_response_packets.html





