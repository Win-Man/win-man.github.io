---
title: MySQL Binlog(三)——MySQL 复制流程详解
date: 2019-12-01 16:41:13
tags:
- MySQL
- Binlog
categories:
- MySQL
---
# MySQL Binlog(三)——MySQL 复制流程详解

## 前言

上一篇文章讲了MySQL协议的基础内容，在本文中我们通过分析MySQL复制流程看看MySQL协议在复制中的使用。

## MySQL复制流程概述

按照复制原理中描述的，在备库上提交change master to请求主备数据连接以后

1. TCP连接阶段：备库和主库之间会建立TCP链接（TCP/IP的三次握手建立连接过程不在本文讨论范畴，这里不详细描述）
2. 用户认证阶段：主库会对备库的用户名密码进行认证，进行MySQL应用层的三次握手
3. register_slave注册slave阶段：备库通过register_slave将自己注册到主库上（该步骤可忽略），以便master在show slave host时可以查看到哪些slave连接上来了
4. request_dump请求binlog阶段：备库通过request_dump，请求从主库指定的二进制日志和文件偏移量开始获取所有的二进制日志。
5. master发送Binlog Event阶段：Master启动binlog dump线程，从指定的二进制日志的对应文件偏移量开始向备库的IO线程发送二进制日志Event。（该步骤在主库上就是读取文件和网络发送的过程，本文档中不详细描述）
6. 备库接收存储Relaylog阶段：备库的IO线程持续接收Event并将Event存储relay log中。具体Event的存储格式以及各种类型的字段在Binlog Event的存储格式请参考后续文章。

接下来的几节将详细对以上2，3，4步的网络协议字节格式、及可能取值等方面进行详细描述

## 用户认证过程详解

### 用户认证时序图

```sequence
Master->Slave:Server Greeting 开始MySQL握手协议
Slave->Master:Login Request 进行账号密码验证登陆
Master->Slave:Response OK 账号密码验证通过
```

如图所示，

1. master先发送了greeting网络包；
2. slave接收到该网络包以后提交帐号密码进行验证；
3. master接收到用户名密码，验证通过后会通过Response OK 返回表示认证通过。

> 普通的客户端连接MySQL同样需要经历这三次握手协议。



### 异常情况

* 第1步如果slave没有接收到greeting网络包，在超时等待后会报错退出
* 第2步，如果master没有接收到slave提交的帐号密码，此时，可以在master数据库中看到类似于如下信息：

![](https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20191201164105.png)

可以看到当前数据库中会有一个连接连上来，但是在User字段显示是的unauthenticated user，这就是表示连接已经建立，但是用户认证还没有完成。该线程的状态是Receiving from client，表示正在等待client发送用户认证的数据信息。超时等待后，master也会报错并中止对应的线程.

* 第3步，如果master密码认证错误，会直接发送一个错误包，slave和Master都需要中止这个连接。

接下来几节将详细描述用户认证时序图中1，2，3三个网络包的具体字节分段含义。

### Server Greeting步骤

#### 相关代码位置

```
sql/rpl_slave.cc文件中的handle_slave_io() 
=> sql/rpl_slave.cc文件中的safe_connect()函数 
=> sql/rpl_slave.cc文件中的connect_to_master()函数
=> sql-common/client.c文件中的CLI_MYSQL_REAL_CONNECT()函数part1
```



#### Greeting数据包格式

| 类型 | 名字 | 描述 |
| ----- | ----- | ----- |
| int<3> | payload_length | 数据包中有效信息的长度，不包括数据包头部4个字节的长度 |
| int<1> | sequence_id | 数据包序列号 |
| int<1> | 协议版本号 | 表示当前连接使用的协议版本 |
| string< null-terminated > | 服务器版本信息 | 表示当前server端服务器版本 |
| int<4> | thread_id | 表示当前连接，在server上的线程ID |
| string< null-terminated > | scramble data part1 | 用于密码校验的随机数部分一 |
| int<2> | | 无意义 |
| int<1> | server_charset | 服务器的字符集 |
| int<2> | server_status | 服务器状态 |
| int<2> | server_capability | 服务器权能标志 |
| int<1> | pkt_scramble_length | |
| string[10] | | 无意义 |
| string< null-terminated > | scramble data part2 | 用于密码校验的随机数部分二 |
| string< null-terminated > | scramble plugin name | 用于密码校验的插件名称 |


#### 数据包解析示例

数据包示例：

```
4e 00 00 数据包长度 //88
00 数据包序号 //0
0a 协议版本号//10
35 2e 37 2e 31 39 2d 6c 6f 67 00 服务器版本信息 //5.7.19-log
72 00 00 00 线程ID //114
7e 1a 15 11 07 0a 7e 1c 00 scramble data part1
ff f7 
53 服务器字符集 //
02 00 服务器状态
ff 81 服务器权能标志
15 pkt_scramble_length
00 00 00 00 00 00 00 00 00 00 无用
0b 2c 62 79 75 7a 3d 01 61 25 6f 22 00 scramble data part2
6d 79 73 71 6c 5f 6e 61 74 69 76 65 5f 70 61 73 73 77 6f 72 64 00 scramble插件名字 //mysql_native_password
```

各分段含义解释：

* 数据包长度：MySQL协议中固定格式，前三个字节表示数据包长度，这个长度不包括固定格式的4个字节
* 数据包序号：MySQL协议中固定格式，用一个字节表示数据包的序号
* 协议版本号：MySQL协议的版本序号
* 服务器版本信息：服务器的版本，这边为5.7.19-log
* 线程ID：表示master分配给该连接的线程的ID号，与show processlist中的thread id对应
* scramble data part1:master 生成的随机数，用于加密密码
* 服务器字符集：表示master使用的字符集编码
* 服务器状态:表示服务器状态

在/include/mysql_com.h定义了服务器的状态

```
#define SERVER_STATUS_IN_TRANS     1
#define SERVER_STATUS_AUTOCOMMIT   2	/* Server in auto_commit mode */
#define SERVER_MORE_RESULTS_EXISTS 8    /* Multi query - next query exists */
#define SERVER_QUERY_NO_GOOD_INDEX_USED 16
#define SERVER_QUERY_NO_INDEX_USED      32
#define SERVER_STATUS_CURSOR_EXISTS 64
#define SERVER_STATUS_LAST_ROW_SENT 128
#define SERVER_STATUS_DB_DROPPED        256 /* A database was dropped */
#define SERVER_STATUS_NO_BACKSLASH_ESCAPES 512
#define SERVER_STATUS_METADATA_CHANGED 1024
#define SERVER_QUERY_WAS_SLOW          2048
#define SERVER_PS_OUT_PARAMS            4096
#define SERVER_STATUS_IN_TRANS_READONLY 8192
#define SERVER_SESSION_STATE_CHANGED (1UL << 14)
```

* 服务器权能标志：用于与客户端协商协议方式

在/include/mysql_com.h定义

```
#define CLIENT_LONG_PASSWORD	1	/* new more secure passwords */
#define CLIENT_FOUND_ROWS	2	/* Found instead of affected rows */
#define CLIENT_LONG_FLAG	4	/* Get all column flags */
#define CLIENT_CONNECT_WITH_DB	8	/* One can specify db on connect */
#define CLIENT_NO_SCHEMA	16	/* Don't allow database.table.column */
#define CLIENT_COMPRESS	32	/* Can use compression protocol */
#define CLIENT_ODBC	64	/* Odbc client */
#define CLIENT_LOCAL_FILES	128	/* Can use LOAD DATA LOCAL */
#define CLIENT_IGNORE_SPACE	256	/* Ignore spaces before '(' */
#define CLIENT_PROTOCOL_41	512	/* New 4.1 protocol */
#define CLIENT_INTERACTIVE	1024	/* This is an interactive client */
#define CLIENT_SSL              2048	/* Switch to SSL after handshake */
#define CLIENT_IGNORE_SIGPIPE   4096    /* IGNORE sigpipes */
#define CLIENT_TRANSACTIONS	8192	/* Client knows about transactions */
#define CLIENT_RESERVED         16384   /* Old flag for 4.1 protocol  */
#define CLIENT_RESERVED2        32768   /* Old flag for 4.1 authentication */
#define CLIENT_MULTI_STATEMENTS (1UL << 16) /* Enable/disable multi-stmt support */
#define CLIENT_MULTI_RESULTS    (1UL << 17) /* Enable/disable multi-results */
#define CLIENT_PS_MULTI_RESULTS (1UL << 18) /* Multi-results in PS-protocol */
#define CLIENT_PLUGIN_AUTH  (1UL << 19) /* Client supports plugin authentication */
#define CLIENT_CONNECT_ATTRS (1UL << 20) /* Client supports connection attributes */
/* Enable authentication response packet to be larger than 255 bytes. */
#define CLIENT_PLUGIN_AUTH_LENENC_CLIENT_DATA (1UL << 21)
/* Don't close the connection for a connection with expired password. */
#define CLIENT_CAN_HANDLE_EXPIRED_PASSWORDS (1UL << 22)
/**
  Capable of handling server state change information. Its a hint to the
  server to include the state change information in Ok packet.
*/
#define CLIENT_SESSION_TRACK (1UL << 23)
/* Client no longer needs EOF packet */
#define CLIENT_DEPRECATE_EOF (1UL << 24)
#define CLIENT_SSL_VERIFY_SERVER_CERT (1UL << 30)
#define CLIENT_REMEMBER_OPTIONS (1UL << 31)
```

* scramble data part2：与scramble data part1组成为随机数，用于加密密码
* scramble插件名字：指定用于scramble data加密方式的插件,这边为native_mysql_password

### Login Request步骤

本节主要描述slave发送的用户认证的数据包(slave->master)过程

#### 相关代码位置

```
sql/rpl_slave.cc文件中的handle_slave_io() 
=> sql/rpl_slave.cc文件中的safe_connect()函数 
=> sql/rpl_slave.cc文件中的connect_to_master()函数
=> sql-common/client.c文件中的CLI_MYSQL_REAL_CONNECT()函数
=> sql-common/client.c文件中的run_plugin_auth()函数
=> sql-common/client.c文件中的native_password_auth_client()函数
```

#### 数据包格式

| 类型 | 名字 | 描述 |
| ----- | ----- | ----- |
| int<3> | payload_length | 数据包中有效信息的长度，不包括数据包头部4个字节的长度 |
| int<1> | sequence_id | 数据包序列号 |
| int<4> | client_flag | client的标记 |
| int<4> | max_packet_size | 允许发送的最大数据包的大小 |
| int<1> | client_charset | client使用的字符集 |
| string[23] | | 无用 |
| string< null-terminated > | user_name | 登陆的用户名 |
| int<1> | client_capability | |
| string[20] | 加密后的scramble_data | |
| string< null-terminated > | database_name | 要连接的数据库的名称 |
| string< null-terminated > | scramble_plugin_name | 用于密码校验的插件名称 |

#### 数据包解析

数据包示例

```
58 00 00 数据包长度 //98
01 数据包序号 //1
0d a2 2b 00 client flag
01 00 00 00 max_packet_size
08 client使用的字符集编码 //latin1
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 无用
71 62 65 6e 63 68 00 用户名 //qbench
14 client权能标识
f2 4c b6 e9 4c 0d d7 9d 07 30 c9 21 07 4f 23 51 09 a1 af 47 加密之后的scramble data
6d 79 73 71 6c 00 连接的数据库 //mysql
6d 79 73 71 6c 5f 6e 61 74 69 76 65 5f 70 61 73 73 77 6f 72 64 00 加密的插件名称 //mysql_native_password
```



各分段数据含义：

* 数据包长度：MySQL协议中固定格式，前三个字节表示数据包长度，这个长度不包括固定格式的4个字节
* 数据包序号：MySQL协议中固定格式，用一个字节表示数据包的序号
* client flag ：指定客户端发送数据的一些规范，
* max_packet_size:指定客户端发送数据包的最大值，
* client使用的字符集编码：client所使用的字符集编码
* 用户名：用于用户登陆的用户名
* client权能标识：client capability
* 加密之后的scramble data：将用于登陆的密码加密之后得到的结果

参考淘宝内核月报关于native_mysql_password插件加密方式：

> client:
> hash_stage1 = sha1(password) hash_stage2 = sha1(hash_stage1) reply = sha1(scramble, hash_stage2) ^ hash_stage1
> server: (逻辑位于sql/password.c:check_scramble_sha1中, 下文亦有提及)
> // mysql.user表中, 对应user的passwd实际上是hash_stage2 res1 = sha1(scramble, hash_stage2) hash_stage1 = reply ^ res1 hash_stage2_reassured = sha1(hash_stage1) 再根据hash_stage2_reassured == hash_stage2(from mysql.user)是否一致来判定是否合法

* 数据库名称：指定要连接的schema的名称
* 加密插件的名称：进行加密的插件的名称，这边为native_mysql_password



#### 异常情况

在用户认证过程中，master发送给slave一个Greeting包之后，slave发送一个auth包给master，因为我们是模拟IO线程的过程，所以这个auth包的内容是我们自己组织的，在发送将auth包发送给master进行用户认证的过程中，可能会出现一些异常情况，这样会导致master返回一个非OK包，这边列举可能出现的异常情况。

* slave发送的auth包中，用户名和密码错误
master会返回一个ERR包，ERR包中会包含‘ACCESS DENIED’的信息 1045号错误。

* slave发送的auth包中，数据组织格式有问题
master会返回一个EOF包，并中断连接。



### response ok步骤详解

认证没有问题master将直接返回一个OK包。

## register_slave注册slave过程过程详解

### register slave 认证时序图

```sequence
Slave->Master:Request Query 设置master_binlog_checksum（必须）
Master->Slave:Response OK 请求执行成功
Slave->Master:Request Query 设置slave_uuid（非必须）
Master->Slave:Response OK 请求执行成功
Slave->Master:Request Register Slave 发送register slave命令（非必须）
Master->Slave:Response OK 注册slave成功
```

如图所示，注册slave的过程如下

1. slave客户端直接SET @master_binlog_checksum= @@global.binlog_checksum
2. 设置正确，返回Ok Packet
3. 设置其他相关配置，比如slave_uuid等
4. 设置正确，返回Ok Packet
5. slave发送register_slave 注册slave请求
6. 设置正确，返回Ok Packet

> 注册slave步骤主要是为了方便主库能获得slave的IP/Port等信息，第5步可忽略


### 异常情况

* 第1步，第3步如果设置的命令有误，Master将返回Error Packet。
* 第5步，如果填充的register_slave Packet包错误，Master同样将返回Error Packet

接下来几节将详细描述上述第5步网络包的具体字节分段含义。

### Register Slave 步骤

#### 相关代码位置

```
sql/rpl_slave.cc文件中的handle_slave_io()函数
==>sql/rpl_slave.cc文件中的register_slave_on_master()函数
```



#### 数据包格式

| 类型 | 名字 | 描述 |
| ----- | ----- | ----- |
| int<3> | payload_length | 数据包中有效信息的长度，不包括数据包头部4个字节的长度 |
| int<1> | sequence_id | 数据包序列号 |
| int<1> | command_type | 发送的请求的类型 |
| int<4> | server_id | slave的server_id |
| string< null-terminated > | report_host | |
| string< null-terminated > | report_user | |
| string< null-terninated > | report_password | |
| int<2> | port | 端口号 |
| string[8] | | 无用 |

#### 数据包解析

数据包示例：

```
12 00 00 数据包长度
00 数据包序号
15 发送的command类型 0x15表示的是COM_REGISTER_SLAVE类型的命令
0f 27 00 00 slave的server_id
00 report host
00 report user
00 report password
ea 0c 端口号
00 00 00 00 00 00 00 00 无用
```

各分段数据含义：

* 数据包长度：MySQL协议中固定格式，前三个字节表示数据包长度，这个长度不包括固定格式的4个字节
* 数据包序号：MySQL协议中固定格式，用一个字节表示数据包的序号
* COMMAND类型：表示发送的请求的类型

在my_command.h文件中定义了command的类型：

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
}
```

* report host:指定report的IP地址
* report user:指定report的user
* report password：指定report的账户的密码
* 端口号：指定连接的端口号，等于change master to语句中的port选项

#### 异常情况

如果填充的register_slave Packet包错误，Master同样将返回Error Packet

## request dump请求binlog过程详解

### request dump 请求时序图

```sequence
Slave->Master:Request Send Binlog 请求发送binlog
Master->Slave:Rotate Event 主库发送一个Rotate Event内容
Master->Slave:Format Description Event 主库发送一个Format Description Event内容
Master->Slave:··················Binlog Event
Master->Slave:··················Binlog Event
Master->Slave:··················Binlog Event
```

如图所示，请求binlog的过程如下

1. slave客户端发送request dump请求，请求中包含binlog dump开始发送二进制文件名称和偏移量
2. master接收到请求，位置正确的话，会先发送Rotate Event表示接下来的二进制日志将从指定的二进制文件和偏移量开始发送
3. master紧接着会从指定的二进制文件头读取Format Description Event 并发送给slave，用于slave 确认各个event的header长度等信息。
4. master从二进制文件和偏移量开始读取二进制数据并通过网络发给Slave

异常情况下：

* 第1步，如果slave请求的二进制文件或者偏移量不正确，master将返回ERR Packet

### Request dump 步骤
#### 相关代码位置

```
sql/rpl_slave.cc文件中的handle_slave_io()函数
==>sql/rpl_slave.cc文件中的request_dump()函数
```

#### 数据包格式

| 类型 | 名字 | 描述 |
| ----- | ----- | ----- |
| int<3> | payload_length | 数据包中有效信息的长度，不包括数据包头部4个字节的长度 |
| int<1> | sequence_id | 数据包序列号 |
| int<1> | command_type | 发送的请求的类型 |
| int<4> | binlog_pos | 请求的binlog位置点 |
| int<2> | binlog_flag | |
| int<4> | server_id | slave的server_id |
| string< reset-of_packet > | binlog_file_name | 请求的binlog文件名 |

#### 数据包解析

```
1b 00 00 数据包长度
00 数据包序号
12 发送的command类型 0x12表示的是COM_BINLOG_DUMP
9a 00 00 00 binlog位置点
00 00 binlog flag
0f 27 00 00 slave_server_id
6d 79 73 71 6c 2d 62 69 6e 2e 30 30 30 30 30 31binlog文件名
```

* 数据包长度：MySQL协议中固定格式，前三个字节表示数据包长度，这个长度不包括固定格式的4个字节
* 数据包序号：MySQL协议中固定格式，用一个字节表示数据包的序号
* COMMAND类型：表示发送的请求的类型
* binlog位置点：表示请求的binlog位置
* binlog flag：
* slave_server_id：表示slave的server_id
* binlog文件名：表示请求的binlog的文件名

#### 异常情况

* binlog 请求的位置点大于对应binlog_file中最大的位置点
Master返回一个ERR数据包，Err Packet格式参见“3.3.2 ”

示例数据包：

```
4f 00 00 数据包长度
01 数据包序号
ff ERR数据包标识
d4 04 error code
23 48 59 30 30 30 43 6c 69 65 6e 74 20 72 65 71 75 65 73 74 65 64 20 6d 61 73 74 65 72 20 74 6f 20 73 74 61 72 74 20 72 65 70 6c 69 63 61 74 69 6f 6e 20 66 72 6f 6d 20 70 6f 73 69 74 69 6f 6e 20 3e 20 66 69 6c 65 20 73 69 7a 65 错误信息
```

各分段数据包含义：

 * 错误号为：1236
 * 错误信息为：Client requested master to start replication from position>filesize

* binlog 请求的位置点不是event起始位置点

Master返回一个ERR数据包，Err Packet格式参见上一篇文章中ERR_Packet部分内容。

数据包示例：

```
e1 00 00 数据包长度
03 数据包序号
ff ERR数据包标识
d4 04 error code
23 48 59 30 30 30 62 6f 67 75 73 20 64 61 74 61 20 69 6e 20 6c 6f 67 20 65 76 65 6e 74 3b 20 74 68 65 20 66 69 72 73 74 20 65 76 65 6e 74 20 27 6d 79 73 71 6c 2d 62 69 6e 2e 30 30 30 30 30 31 27 20 61 74 20 31 36 30 2c 20 74 68 65 20 6c 61 73 74 20 65 76 65 6e 74 20 72 65 61 64 20 66 72 6f 6d 20 27 2f 6f 70 74 2f 6d 79 73 71 6c 2f 64 61 74 61 2f 62 69 6e 6c 6f 67 2f 6d 79 73 71 6c 2d 62 69 6e 2e 30 30 30 30 30 31 27 20 61 74 20 31 32 33 2c 20 74 68 65 20 6c 61 73 74 20 62 79 74 65 20 72 65 61 64 20 66 72 6f 6d 20 27 2f 6f 70 74 2f 6d 79 73 71 6c 2f 64 61 74 61 2f 62 69 6e 6c 6f 67 2f 6d 79 73 71 6c 2d 62 69 6e 2e 30 30 30 30 30 31 27 20 61 74 20 31 37 39 2e 错误信息
```

各分段含义：

 * 错误号为：1236
 * 错误信息为：bogusdatain log event;the first event'mysql-bin.000001'at160,the last event read from'/opt/mysql/data/binlog/mysql-bin.000001'at123,the last byte read from'/opt/mysql/data/binlog/mysql-bin.000001'at179.

* binlog 请求的binlog filename 不存在

Master返回一个ERR数据包

数据包示例：

```
44 00 00 数据包长度
01 数据包序号
ff ERR数据包标识
d4 04 error code
23 48 59 30 30 30 43 6f 75 6c 64 20 6e 6f 74 20 66 69 6e 64 20 66 69 72 73 74 20 6c 6f 67 20 66 69 6c 65 20 6e 61 6d 65 20 69 6e 20 62 69 6e 61 72 79 20 6c 6f 67 20 69 6e 64 65 78 20 66 69 6c 65 错误信息
```

各分段数据含义：

 * 错误号为：1236
 * 错误信息为：Could not find first log filename in binary log index file


## 整体流程回顾
最后我们再整体来看一下MySQL复制链接在建立连接的完整过程

* Master与Slave建立TCP/IP连接
* Master发送第一个MySQL协议的Greeting包给Slave
* Slave根据发送的数据包，加密自己的账户及密码等信息，发送给Master
* Master根据Slave发送的auth数据包进行认证，认证成功，则发送一个Response OK数据包给Slave，Slave完成用户登录
* Slave发送一个Query的请求，设置当前连接的master_binlog_checksum为@@global.binlog_checksum这一步是必须的，这是设置校验码
* Master处理完Slave的Query请求之后，发送一个Response OK数据包给Slave
* Slave发送一个Query请求，设置当前连接的slave_uuid为从库的uuid，这一步非必须
* Master处理完Slave的Query请求之后，发送一个Response OK数据包给Slave
* Slave向Master发送一个Register Slave的数据包，进行slave线程的注册
* Master将slave线程注册完成之后，返回一个Response OK数据包
* Slave在注册成功之后，向Master发送Binlog Dump命令，发送请求的binlog的文件名以及pos点
* Master在收到Slave的Binlog Dump命令之后，首先发送一个Rotate Event的内容给Slave，告诉Slave当前发送的binlog的文件名为什么
* Master之后再发送一个Format Description Event内容给Slave，包括版本信息，位置点等信息
* 接着Master按照Slave的Binlog Dump请求中携带的pos点的信息，从该binlog文件偏移位置为pos的地方将Event内容取出来，发送给Slave



## 参考链接

*  MySQL协议官方文档[https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_PROTOCOL.html](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_PROTOCOL.html)
* http://mysql.taobao.org/monthly/2017/08/05/



