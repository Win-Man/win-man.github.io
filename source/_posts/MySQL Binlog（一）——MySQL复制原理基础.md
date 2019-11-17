---
title: MySQL Binlog（一）——MySQL复制原理基础
date: 2019-11-17 09:35:13
tags:
- MySQL
- Binlog
categories:
- MySQL
---
# MySQL Binlog（一）——MySQL复制原理基础

## 前言

复制作为MySQL DBA必备技能之一，从本章开始，我们将开启一个文章系列，从MySQL复制原理、MySQL协议、MySQL字段存储以及MySQL binlog event存储几个方面展开介绍。本系列文章主要参考官方文档以及官方Internals Manual，部分通过查看5.6.34以及5.7.22版本源码补全。

本文作为此系列文章第一篇，介绍一下MySQL复制原理的基础知识。


## 复制原理

如下图所示，MySQL主备复制基于二进制日志binlog。任何数据更改都会写入二进制日志。

数据库管理员搭建主备复制时，只需要在备库change master to指定主库的IP、端口、同步开始的二进制文件和文件偏移量（MySQL 5.6以后支持GTID模式，二进制文件和文件偏移量可以用GTID号集合替换）就可以了。

备库通过IO线程连接主库，接收主库推送过来的二进制日志，并记录到本地的中继日志relaylog；同时也会启动SQL线程将中继日志的数据变更应用到备库本地数据库中.

主库接受到备库IO线程的请求，会专门对该slave启用独立的binlog dump线程，从IO线程指定的二进制文件和文件偏移量开始发送二进制日志；并且在主库有任何新的变更后，在记录到自己的二进制日志的同时也会通过网络推送给备库的IO线程。
![](https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20191117093115.png)

## 复制线程

### Master线程

* binlog dump线程
dump线程的作用是读取主库上二进制日志中的事件。在复制线程处于正常运行状态时，当事务提交的时候，binlog日志sync到磁盘上之后，MySQL会调用signal_update()函数，这个函数的作用是通知binlog dump线程，binlog日志有更新了，dump线程将产生的增量binlog推送到从库的IO线程；在主从之间建议复制连接的时候，从库IO线程将binlog文件名以及位置点（GTID模式下是发送GTID集合）发送给主句dump线程拉取从库所需binlog。
对于一主多从的情况，master上会有多个binlog dump线程。


### Slave线程

* I/O线程：
I/O线程的作用就是拉取主库上的binlog日志，在从库上存贮为relay log日志。方便SQL线程中relay log中重放事务。
I/O线程在与主库建立连接的时候，超过slave_net_timeout时间没有建立连接成功，从库就认为这次连接失败，需要重试连接，重试连接的次数由MASTER_RETRY_COUNT决定。
I/O线程在与主库成功建立连接之后，针对可能主库很长时间都没有更新数据的情况，I/O线程采用了心跳机制，I/O线程在空闲的时候，每隔MASTER_HEARTBEAT_PERIOD时间间隔，I/O线程就向主库发送一个心跳包，测试与主库的连接是否正常。
使用mysqlbinlog工具也可以解析relay log日志。

* SQL线程：
读取relay log，并且重放relay log中的事务，以达到复制的目的。


## 复制格式

binlog的格式可以分为三种，从某种意义上来讲也可以说对应着三种复制模式，使用binlog_format控制binlog的格式，该参数的不同值代表了不同的复制模式，下面以该参数的三个值对复制格式进行简单的阐述：
 * statement：5.1.5之前只支持statement格式（俗称：statement binary replication，缩写：SBR），简单实现了数据同步，但在执行跨裤更新等SQL语句时容易出现主从的数据不一致
 * row：5.1.5及其之后，新增支持row格式（俗称：row binary replication，缩写：RBR），不再是简单记录SQL执行顺序，而是逐行记录了存储引擎的数据如何变更的，主从之间的数据一致性保障得到大幅度提升
 * mixed：5.1.8及其之后新增支持mixed格式（俗称：mixed binary replication，缩写：MBR），本质上是让MySQL Server自行根据SQL语句的不同来判断是否需要使用row格式，当出现可能造成主从数据不一致的SQL时（例如：用户自定义函数，跨库SQL等），binlog自动转为row格式记录，否则默认使用statement格式记录
## MySQL binlog

### 什么是binlog

二进制日志（binary log简称binlog）是MySQL中的日志文件，会记录MySQL数据库执行的更改操作。但是只记录更改操作，如SELECT、SHOW等操作，不会被记录到binlog中。binlog对于MySQL而言是很重要的日志文件，它可以有以下功能：

* 搭建复制

MySQL的复制完全依赖于binlog搭建，复制的原理本质上就是从库从主库上拉取主库的二进制日志，然后将拉取到的日志在从库上进行重放，实现数据的同步。

* 数据恢复

一般的备份方案，都是特定的时刻对数据库进行备份，没有对数据库进行实时的备份。如果数据库在两个备份时间点之间出现故障，那么恢复数据的时候，最多只能恢复到最近一个备份时间点。但是binlog日志可以基于时间点或者位置点进行数据恢复。在上述故障情况的时候，可以拿最近一个时间点的备份+binlog将数据恢复到故障时刻。

* 审计

用户可以通过二进制日志中的信息来进行审计，判断是否有对数据局进行注入的攻击。



### binlog文件

binlog文件主要包括：

```
mysql-bin.00000x
mysql-bin.index
```

`mysql-bin.index`记录了数据库中还未被purge的binlog文件名。

```
[root@10-10-90-177 binlog]# cat mysql-bin.index
/home/mysql/data/mysqldata1/binlog/mysql-bin.000001
/home/mysql/data/mysqldata1/binlog/mysql-bin.000002
/home/mysql/data/mysqldata1/binlog/mysql-bin.000003
```

`mysql-bin.00000x`是真正的binlog文件 ，x表示binlog文件的序号，这些二进制文件里面的数据是以二进制的形式保存，可以通过特殊的工具(mysqlbinlog)进行解析，获取人类可读的信息。


### binlog文件组成

binlog文件格式有以下特点：

1. binlog是由event组成，event是binlog的最小组成单元
2. binlog文件头部固定以4个字节开头，这四个字节称为BINLOG_MAGIC(fe 62 69 6e)
3. 每个binlog文件以一个Format_desc类型的event开始
4. 每个binlog文件以一个Rotate类型的event结束（但也有特殊情况，当数据库出现宕机的情况，重新启动数据库会生成一个新的binlog文件，但是宕机之前的最新binlog文件中，不是以ROTATE_EVENT结束的）
5. 在FORMAT_DESCRIPTION_EVENT和ROTATE_EVENT之间是各种不同event，每个event代表Master上不同的操作。



* BINLOG_MAGIC

每个binlog文件以固定的4个字节（fe 62 69 6e）开始，以表示是一个binlog文件，在Linux环境下，我们可以通过`hexdump`命令查看binlog文件的字节组成。
![](https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20191117093140.png)

* event

我们有两种方式可以查看，binlog文件中的event组成方式。一种是在MySQL数据库中通过`SHOW BINLOG EVENTS IN 'binlog-file-name'`的方式查看，还有一种就是通过mysqlbinlog工具查看binlog文件的组成。

 * 在数据库中查看

```
root@localhost : (none) 10:50:48> show binlog events in 'mysql-bin.000004';
+------------------+-----+-------------+-----------+-------------+-----------------------------------------------------------+
| Log_name        | Pos | Event_type  | Server_id | End_log_pos | Info                                                      |
+------------------+-----+-------------+-----------+-------------+-----------------------------------------------------------+
| mysql-bin.000004 |  4 | Format_desc |  3306114 |        120 | Server ver: 5.6.34-log, Binlog ver: 4                    |
| mysql-bin.000004 | 120 | Query      |  3306114 |        201 | BEGIN                                                    |
| mysql-bin.000004 | 201 | Rows_query  |  3306114 |        265 | # insert into test1(`name`) values('woqu')                |
| mysql-bin.000004 | 265 | Table_map  |  3306114 |        320 | table_id: 70 (gangshen.test1)                            |
| mysql-bin.000004 | 320 | Write_rows  |  3306114 |        365 | table_id: 70 flags: STMT_END_F                            |
| mysql-bin.000004 | 365 | Xid        |  3306114 |        396 | COMMIT /* xid=24 */                                      |
| mysql-bin.000004 | 396 | Query      |  3306114 |        477 | BEGIN                                                    |
| mysql-bin.000004 | 477 | Rows_query  |  3306114 |        556 | # update test1 set name='woqu-change' where name = 'woqu' |
| mysql-bin.000004 | 556 | Table_map  |  3306114 |        611 | table_id: 70 (gangshen.test1)                            |
| mysql-bin.000004 | 611 | Update_rows |  3306114 |        674 | table_id: 70 flags: STMT_END_F                            |
| mysql-bin.000004 | 674 | Xid        |  3306114 |        705 | COMMIT /* xid=27 */                                      |
| mysql-bin.000004 | 705 | Query      |  3306114 |        786 | BEGIN                                                    |
| mysql-bin.000004 | 786 | Rows_query  |  3306114 |        854 | # delete from test1 where name = 'woqu-change'            |
| mysql-bin.000004 | 854 | Table_map  |  3306114 |        909 | table_id: 70 (gangshen.test1)                            |
| mysql-bin.000004 | 909 | Delete_rows |  3306114 |        961 | table_id: 70 flags: STMT_END_F                            |
| mysql-bin.000004 | 961 | Xid        |  3306114 |        992 | COMMIT /* xid=28 */                                      |
| mysql-bin.000004 | 992 | Rotate      |  3306114 |        1039 | mysql-bin.000005;pos=4                                    |
+------------------+-----+-------------+-----------+-------------+-----------------------------------------------------------+
17 rows in set (0.00 sec)
```

可以看到`mysql-bin.000004`文件中，是以一个Format_desc类型的event开头，以一个Rotate类型的event结尾，中间是各种类型event，这是看binlog文件中event的组成，如果想要查看更详细的event信息，可以通过mysqlbinlog工具来查看

 * mysqlbinlog工具查看

```
/*!40019 SET @@session.max_insert_delayed_threads=0*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 4
#171205 10:35:43 server id 3306114  end_log_pos 120 CRC32 0x7d7e496c    Start: binlog v 4, server v 5.6.34-log created 171205 10:35:43
# at 120
#171205 10:49:25 server id 3306114  end_log_pos 201 CRC32 0x24d309ef    Query  thread_id=2    exec_time=0    error_code=0
SET TIMESTAMP=1512442165/*!*/;
SET @@session.pseudo_thread_id=2/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=1075838976/*!*/;
SET @@session.auto_increment_increment=2, @@session.auto_increment_offset=2/*!*/;
/*!\C utf8 *//*!*/;
SET @@session.character_set_client=33,@@session.collation_connection=33,@@session.collation_server=83/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
BEGIN
/*!*/;
# at 201
#171205 10:49:25 server id 3306114  end_log_pos 265 CRC32 0xef0e9a3a    Rows_query
# insert into test1(`name`) values('woqu')
# at 265
#171205 10:49:25 server id 3306114  end_log_pos 320 CRC32 0xf86a076e    Table_map: `gangshen`.`test1` mapped to number 70
# at 320
#171205 10:49:25 server id 3306114  end_log_pos 365 CRC32 0x653b8de4    Write_rows: table id 70 flags: STMT_END_F
### INSERT INTO `gangshen`.`test1`
### SET
###  @1=4 /* INT meta=0 nullable=0 is_null=0 */
###  @2='woqu' /* VARSTRING(60) meta=60 nullable=1 is_null=0 */
# at 365
#171205 10:49:25 server id 3306114  end_log_pos 396 CRC32 0x53ca8e9d    Xid = 24
COMMIT/*!*/;
............
# at 705
#171205 10:50:20 server id 3306114  end_log_pos 786 CRC32 0x815b7a14    Query  thread_id=2    exec_time=0    error_code=0
SET TIMESTAMP=1512442220/*!*/;
BEGIN
/*!*/;
# at 786
#171205 10:50:20 server id 3306114  end_log_pos 854 CRC32 0xc265e50d    Rows_query
# delete from test1 where name = 'woqu-change'
# at 854
#171205 10:50:20 server id 3306114  end_log_pos 909 CRC32 0x3a62ecaf    Table_map: `gangshen`.`test1` mapped to number 70
# at 909
#171205 10:50:20 server id 3306114  end_log_pos 961 CRC32 0x2bf4c689    Delete_rows: table id 70 flags: STMT_END_F
### DELETE FROM `gangshen`.`test1`
### WHERE
###  @1=4 /* INT meta=0 nullable=0 is_null=0 */
###  @2='woqu-change' /* VARSTRING(60) meta=60 nullable=1 is_null=0 */
# at 961
#171205 10:50:20 server id 3306114  end_log_pos 992 CRC32 0x412de6a3    Xid = 28
COMMIT/*!*/;
# at 992
#171205 10:50:23 server id 3306114  end_log_pos 1039 CRC32 0x8372f001  Rotate to mysql-bin.000005  pos: 4
DELIMITER ;
# End of log file
ROLLBACK /* added by mysqlbinlog */;
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```

可以看到第一个event是FORMAT_DESCRIPTION_EVENT,记录了binlog的版本为v4。最后一个event，是ROTATE_EVENT，它记录了下一个binlog文件的文件名以及对应的位置点。


## 参考链接

* 官方文档复制章节：https://dev.mysql.com/doc/refman/5.7/en/replication.html

