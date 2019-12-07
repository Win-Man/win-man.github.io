---
title: MySQL Binlog(十一)——MySQL binlog event解析实例
date: 2019-12-07 16:58:13
tags:
- MySQL
- Binlog
categories:
- MySQL
---

# MySQL Binlog(十一)——MySQL binlog event解析实例

## 前言

最后一篇文章是基于前面的知识，进行实际binlog的解析，看看常见的insert、update、delete语句在binlog中存储的形式。

## 示例：INSERT语句在binlog中event表现形式

下面，我们就看一下一条INSERT语句在会产生哪些类型的event，这些event在数据库中查看是怎么样的，在binlog文件中查看是怎么样的。

### ROW格式下表现形式

以下显示内容是在MySQL-5.6.34版本上，且binlog日志记录格式为ROW模式的情况下。

```
root@localhost : gangshen 09:34:02> show binary logs;
+------------------+-----------+
| Log_name        | File_size |
+------------------+-----------+
| mysql-bin.000001 |      120 |
+------------------+-----------+
1 row in set (0.00 sec)

root@localhost : gangshen 09:34:22> show binlog events in 'mysql-bin.000001';
+------------------+-----+-------------+-----------+-------------+---------------------------------------+
| Log_name        | Pos | Event_type  | Server_id | End_log_pos | Info                                  |
+------------------+-----+-------------+-----------+-------------+---------------------------------------+
| mysql-bin.000001 |  4 | Format_desc |    330619 |        120 | Server ver: 5.6.34-log, Binlog ver: 4 |
+------------------+-----+-------------+-----------+-------------+---------------------------------------+
1 row in set (0.00 sec)

root@localhost : gangshen 09:34:32> insert into test1(`name`) values('woqutech');
Query OK, 1 row affected (0.03 sec)

root@localhost : gangshen 09:36:01> show binlog events in 'mysql-bin.000001';
+------------------+-----+-------------+-----------+-------------+------------------------------------------------+
| Log_name        | Pos | Event_type  | Server_id | End_log_pos | Info                                          |
+------------------+-----+-------------+-----------+-------------+------------------------------------------------+
| mysql-bin.000001 |  4 | Format_desc |    330619 |        120 | Server ver: 5.6.34-log, Binlog ver: 4          |
| mysql-bin.000001 | 120 | Query      |    330619 |        201 | BEGIN                                          |
| mysql-bin.000001 | 201 | Rows_query  |    330619 |        269 | # insert into test1(`name`) values('woqutech') |
| mysql-bin.000001 | 269 | Table_map  |    330619 |        324 | table_id: 70 (gangshen.test1)                  |
| mysql-bin.000001 | 324 | Write_rows  |    330619 |        373 | table_id: 70 flags: STMT_END_F                |
| mysql-bin.000001 | 373 | Xid        |    330619 |        404 | COMMIT /* xid=13 */                            |
+------------------+-----+-------------+-----------+-------------+------------------------------------------------+
6 rows in set (0.00 sec)
```

从数据库中，我们可以看到`insert into test1(`name`) values('woqutech');`语句，在binlog日志文件中转换成了5个event存储，分别为Query,Rows_query,Table_map,Write_rows,Xid类型的event，这些event中，Query event表示一个更新语句的开始，Rows_query event记录了更新语句的语句内容，Table_map event中记录了insert语句操作的表的信息，Write_rows event记录了真实更新的记录的内容，最后一个Xid event中表示COMMIT操作。

在数据库中查看binlog文件内容可以很直观的看到，event的类型以及主要内容，那我们接着来看看，在binlog文件中，通过mysqlbinlog工具可以看到的具体内容是什么样的。

可以看到通过mysqlbinlog工具查看binlog文件，可以看到每个event中更加详细的内容。

### STATEMENT格式下表现形式
以下显示内容是在MySQL-5.6.34版本上，且binlog日志记录格式为STATEMENT模式的情况下。

```
root@localhost : gangshen 04:31:35> show binary logs;
+------------------+-----------+
| Log_name        | File_size |
+------------------+-----------+
| mysql-bin.000001 |      143 |
| mysql-bin.000002 |      120 |
+------------------+-----------+
2 rows in set (0.00 sec)

root@localhost : gangshen 04:31:48> show binlog events in 'mysql-bin.000002';
+------------------+-----+-------------+-----------+-------------+---------------------------------------+
| Log_name        | Pos | Event_type  | Server_id | End_log_pos | Info                                  |
+------------------+-----+-------------+-----------+-------------+---------------------------------------+
| mysql-bin.000002 |  4 | Format_desc |    330619 |        120 | Server ver: 5.6.34-log, Binlog ver: 4 |
+------------------+-----+-------------+-----------+-------------+---------------------------------------+
1 row in set (0.00 sec)

root@localhost : gangshen 04:32:02> insert into test1(`name`) values('woqu112233');
Query OK, 1 row affected (0.05 sec)

root@localhost : gangshen 04:32:29> show binlog events in 'mysql-bin.000002';
+------------------+-----+-------------+-----------+-------------+----------------------------------------------------------------+
| Log_name        | Pos | Event_type  | Server_id | End_log_pos | Info                                                          |
+------------------+-----+-------------+-----------+-------------+----------------------------------------------------------------+
| mysql-bin.000002 |  4 | Format_desc |    330619 |        120 | Server ver: 5.6.34-log, Binlog ver: 4                          |
| mysql-bin.000002 | 120 | Query      |    330619 |        212 | BEGIN                                                          |
| mysql-bin.000002 | 212 | Intvar      |    330619 |        244 | INSERT_ID=8                                                    |
| mysql-bin.000002 | 244 | Query      |    330619 |        377 | use `gangshen`; insert into test1(`name`) values('woqu112233') |
| mysql-bin.000002 | 377 | Xid        |    330619 |        408 | COMMIT /* xid=17 */                                            |
+------------------+-----+-------------+-----------+-------------+----------------------------------------------------------------+
5 rows in set (0.00 sec)
```

从数据库中我们可以看到`insert into test1(`name`) values('woqu112233');`这个insert语句转换成了4个event，分别是Query,Intvar,Quert,Xid4个event。第一个Query的event表示语句开始执行，第二个Intvar的event，是因为表结构中的id字段定义的是auto_increment，这个Intvar event是对自增的主键生成主键值的，接着的Query的event就是真实执行的语句了，最后一个Xid表示语句的提交。
在数据库中查看binlog文件内容可以很直观的看到，event的类型以及主要内容，那我们接着来看看，在binlog文件中，通过mysqlbinlog工具可以看到的具体内容是什么样的。

## 示例：UPDATE语句在binlog中event表现形式

下面，我们就看一下一条UPDATE语句在会产生哪些类型的event，这些event在数据库中查看是怎么样的，在binlog文件中查看是怎么样的。

### ROW格式下表现形式

以下显示内容是在MySQL-5.6.34版本上，且binlog日志记录格式为ROW模式的情况下。

```
mysql> show binary logs;
+------------------+-----------+
| Log_name        | File_size |
+------------------+-----------+
| mysql-bin.000001 |      120 |
+------------------+-----------+
1 row in set (0.00 sec)

mysql> show binlog events in 'mysql-bin.000001';
+------------------+-----+-------------+-----------+-------------+---------------------------------------------+
| Log_name        | Pos | Event_type  | Server_id | End_log_pos | Info                                        |
+------------------+-----+-------------+-----------+-------------+---------------------------------------------+
| mysql-bin.000001 |  4 | Format_desc |      9999 |        120 | Server ver: 5.6.34-debug-log, Binlog ver: 4 |
+------------------+-----+-------------+-----------+-------------+---------------------------------------------+
1 row in set (0.00 sec)

mysql> update test1 set name='woqutech_new' where name='woqutech';
Query OK, 1 row affected (0.01 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> show binlog events in 'mysql-bin.000001';
+------------------+-----+-------------+-----------+-------------+--------------------------------------------------------------+
| Log_name        | Pos | Event_type  | Server_id | End_log_pos | Info                                                        |
+------------------+-----+-------------+-----------+-------------+--------------------------------------------------------------+
| mysql-bin.000001 |  4 | Format_desc |      9999 |        120 | Server ver: 5.6.34-debug-log, Binlog ver: 4                  |
| mysql-bin.000001 | 120 | Query      |      9999 |        196 | BEGIN                                                        |
| mysql-bin.000001 | 196 | Rows_query  |      9999 |        278 | # update test1 set name='woqutech_new' where name='woqutech' |
| mysql-bin.000001 | 278 | Table_map  |      9999 |        333 | table_id: 70 (gangshen.test1)                                |
| mysql-bin.000001 | 333 | Update_rows |      9999 |        401 | table_id: 70 flags: STMT_END_F                              |
| mysql-bin.000001 | 401 | Xid        |      9999 |        432 | COMMIT /* xid=16 */                                          |
+------------------+-----+-------------+-----------+-------------+--------------------------------------------------------------+
6 rows in set (0.00 sec)
```

从数据库中，我们可以看到`update test1 set name='woqutech_new' where name='woqutech';`语句，在binlog日志文件中转换成了5个event存储，分别为Query,Rows_query,Table_map,Update_rows,Xid类型的event，这些event中，Query event表示一个更新语句的开始，Rows_query event记录了更新语句的语句内容，Table_map event中记录了insert语句操作的表的信息，Update_rows event记录了真实更新的记录的内容，最后一个Xid event中表示COMMIT操作。

在数据库中查看binlog文件内容可以很直观的看到，event的类型以及主要内容，那我们接着来看看，在binlog文件中，通过mysqlbinlog工具可以看到的具体内容是什么样的。

可以看到通过mysqlbinlog工具查看binlog文件，可以看到每个event中更加详细的内容。


### STATEMENT格式下表现形式
以下显示内容是在MySQL-5.6.34版本上，且binlog日志记录格式为STATEMENT模式的情况下。

```
mysql> show binary logs;
+------------------+-----------+
| Log_name        | File_size |
+------------------+-----------+
| mysql-bin.000001 |      120 |
+------------------+-----------+
1 row in set (0.00 sec)

mysql> show binlog events in 'mysql-bin.000001';
+------------------+-----+-------------+-----------+-------------+---------------------------------------------+
| Log_name        | Pos | Event_type  | Server_id | End_log_pos | Info                                        |
+------------------+-----+-------------+-----------+-------------+---------------------------------------------+
| mysql-bin.000001 |  4 | Format_desc |      9999 |        120 | Server ver: 5.6.34-debug-log, Binlog ver: 4 |
+------------------+-----+-------------+-----------+-------------+---------------------------------------------+
1 row in set (0.00 sec)

mysql> update test1 set name='woqutech_new' where name='woqutech';
Query OK, 1 row affected (0.01 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> show binlog events in 'mysql-bin.000001';
+------------------+-----+-------------+-----------+-------------+----------------------------------------------------------------------------+
| Log_name        | Pos | Event_type  | Server_id | End_log_pos | Info                                                                      |
+------------------+-----+-------------+-----------+-------------+----------------------------------------------------------------------------+
| mysql-bin.000001 |  4 | Format_desc |      9999 |        120 | Server ver: 5.6.34-debug-log, Binlog ver: 4                                |
| mysql-bin.000001 | 120 | Query      |      9999 |        207 | BEGIN                                                                      |
| mysql-bin.000001 | 207 | Query      |      9999 |        347 | use `gangshen`; update test1 set name='woqutech_new' where name='woqutech' |
| mysql-bin.000001 | 347 | Xid        |      9999 |        378 | COMMIT /* xid=46 */                                                        |
+------------------+-----+-------------+-----------+-------------+----------------------------------------------------------------------------+
4 rows in set (0.00 sec)
```

从数据库中我们可以看到`update test1 set name='woqutech_new' where name='woqutech';`这个insert语句转换成了3个event，分别是Query,Quert,Xid4个event。第一个Query的event表示语句开始执行，接着的Query的event就是真实执行的语句了，最后一个Xid表示语句的提交。
在数据库中查看binlog文件内容可以很直观的看到，event的类型以及主要内容，那我们接着来看看，在binlog文件中，通过mysqlbinlog工具可以看到的具体内容是什么样的。


## 示例：DELETE语句在binlog中event表现形式

下面，我们就看一下一条DELETE语句在会产生哪些类型的event，这些event在数据库中查看是怎么样的，在binlog文件中查看是怎么样的。

### ROW格式下表现形式

以下显示内容是在MySQL-5.6.34版本上，且binlog日志记录格式为ROW模式的情况下。

```
root@localhost : gangshen 09:34:02> show binary logs;
+------------------+-----------+
| Log_name        | File_size |
+------------------+-----------+
| mysql-bin.000001 |      120 |
+------------------+-----------+
1 row in set (0.00 sec)

root@localhost : gangshen 09:34:22> show binlog events in 'mysql-bin.000001';
+------------------+-----+-------------+-----------+-------------+---------------------------------------+
| Log_name        | Pos | Event_type  | Server_id | End_log_pos | Info                                  |
+------------------+-----+-------------+-----------+-------------+---------------------------------------+
| mysql-bin.000001 |  4 | Format_desc |    330619 |        120 | Server ver: 5.6.34-log, Binlog ver: 4 |
+------------------+-----+-------------+-----------+-------------+---------------------------------------+
1 row in set (0.00 sec)

mysql> delete from test1 where id = 1;
Query OK, 1 row affected (0.02 sec)

mysql> show binlog events in 'mysql-bin.000001';
+------------------+-----+-------------+-----------+-------------+---------------------------------------------+
| Log_name        | Pos | Event_type  | Server_id | End_log_pos | Info                                        |
+------------------+-----+-------------+-----------+-------------+---------------------------------------------+
| mysql-bin.000001 |  4 | Format_desc |      9999 |        120 | Server ver: 5.6.34-debug-log, Binlog ver: 4 |
| mysql-bin.000001 | 120 | Query      |      9999 |        196 | BEGIN                                      |
| mysql-bin.000001 | 196 | Rows_query  |      9999 |        250 | # delete from test1 where id = 1            |
| mysql-bin.000001 | 250 | Table_map  |      9999 |        305 | table_id: 70 (gangshen.test1)              |
| mysql-bin.000001 | 305 | Delete_rows |      9999 |        358 | table_id: 70 flags: STMT_END_F              |
| mysql-bin.000001 | 358 | Xid        |      9999 |        389 | COMMIT /* xid=27 */                        |
+------------------+-----+-------------+-----------+-------------+---------------------------------------------+
6 rows in set (0.00 sec)

```

从数据库中，我们可以看到`delete from test1 where id = 1;`语句，在binlog日志文件中转换成了5个event存储，分别为Query,Rows_query,Table_map,Delete_rows,Xid类型的event，这些event中，Query event表示一个更新语句的开始，Rows_query event记录了更新语句的语句内容，Table_map event中记录了insert语句操作的表的信息，Delete_rows event记录了真实更新的记录的内容，最后一个Xid event中表示COMMIT操作。

在数据库中查看binlog文件内容可以很直观的看到，event的类型以及主要内容，那我们接着来看看，在binlog文件中，通过mysqlbinlog工具可以看到的具体内容是什么样的。

可以看到通过mysqlbinlog工具查看binlog文件，可以看到每个event中更加详细的内容。

### STATEMENT格式下表现形式
以下显示内容是在MySQL-5.6.34版本上，且binlog日志记录格式为STATEMENT模式的情况下。

```
mysql> show binary logs;
+------------------+-----------+
| Log_name        | File_size |
+------------------+-----------+
| mysql-bin.000001 |      120 |
+------------------+-----------+
1 row in set (0.00 sec)

mysql> show binlog events in 'mysql-bin.000001';
+------------------+-----+-------------+-----------+-------------+---------------------------------------------+
| Log_name        | Pos | Event_type  | Server_id | End_log_pos | Info                                        |
+------------------+-----+-------------+-----------+-------------+---------------------------------------------+
| mysql-bin.000001 |  4 | Format_desc |      9999 |        120 | Server ver: 5.6.34-debug-log, Binlog ver: 4 |
+------------------+-----+-------------+-----------+-------------+---------------------------------------------+
1 row in set (0.00 sec)

mysql> delete from test1 where name = 'woqutech_new';
Query OK, 1 row affected (0.01 sec)

mysql> show binlog events in 'mysql-bin.000001';
+------------------+-----+-------------+-----------+-------------+---------------------------------------------------------------+
| Log_name        | Pos | Event_type  | Server_id | End_log_pos | Info                                                          |
+------------------+-----+-------------+-----------+-------------+---------------------------------------------------------------+
| mysql-bin.000001 |  4 | Format_desc |      9999 |        120 | Server ver: 5.6.34-debug-log, Binlog ver: 4                  |
| mysql-bin.000001 | 120 | Query      |      9999 |        207 | BEGIN                                                        |
| mysql-bin.000001 | 207 | Query      |      9999 |        334 | use `gangshen`; delete from test1 where name = 'woqutech_new' |
| mysql-bin.000001 | 334 | Xid        |      9999 |        365 | COMMIT /* xid=58 */                                          |
+------------------+-----+-------------+-----------+-------------+---------------------------------------------------------------+
4 rows in set (0.00 sec)
```

从数据库中我们可以看到`delete from test1 where name = 'woqutech_new';`这个insert语句转换成了3个event，分别是Query,Quert,Xid4个event。第一个Query的event表示语句开始执行，接着的Query的event就是真实执行的语句了，最后一个Xid表示语句的提交。
在数据库中查看binlog文件内容可以很直观的看到，event的类型以及主要内容，那我们接着来看看，在binlog文件中，通过mysqlbinlog工具可以看到的具体内容是什么样的。

