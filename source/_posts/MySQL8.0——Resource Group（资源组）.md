---
title: MySQL8.0——Resource Group
date: 2018-07-04 17:12:13
tags:
- MySQL
categories:
- MySQL
---
# MySQL8.0——Resource Group（资源组）
## 资源组介绍
### 简介
MySQL是单进程多线程的程序，MySQL线程包括后台线程（Master Thread、IO Thread、Purge Thread等），以及用户线程。在8.0之前，所有线程的优先级都是一样的，并且所有的线程的资源都是共享的。但是在MySQL8.0之后，由于Resource Group特性的引入，我们可以来通过资源组的方式修改线程的**优先级**以及所能使用的**资源**，可以指定不同的线程使用特定的资源。
在目前版本中DBA只能操控CPU资源，并且控制的最小力度为vCPU，即操作系统逻辑CPU核数（可以通过`lscpu`命令查看可控制CPU总数）。
DBA经常会遇到需要执行跑批任务的需求，这种跑批的SQL一般都是很复杂、运行时间长、消耗资源多的SQL。所以很多跑批任务都是在业务低峰期的时候执行，并且在从库上执行，尽可能降低对业务产生影响。但是对于一些数据一致性比较高的跑批任务，需要在主库上执行，在跑批任务运行的过程中很容易影响到其他线程的运行。那么现在Resource Group就是DBA的福音了，我们可以对跑批任务指定运行的资源组，限制任务使用的资源，减少对其他线程的影响。
### 资源组信息查看
* INFORMATION_SCHEMA.RESOURCE_GROUPS

INFORMATION_SCHEMA库下的RESOURCE_GROUPS表中记录了所有定义的资源组的情况:

```
mysql> select * from information_schema.resource_groups;
+---------------------+---------------------+------------------------+----------+-----------------+
| RESOURCE_GROUP_NAME | RESOURCE_GROUP_TYPE | RESOURCE_GROUP_ENABLED | VCPU_IDS | THREAD_PRIORITY |
+---------------------+---------------------+------------------------+----------+-----------------+
| USR_default         | USER                |                      1 | 0-23     |               0 |
| SYS_default         | SYSTEM              |                      1 | 0-23     |               0 |
+---------------------+---------------------+------------------------+----------+-----------------+
2 rows in set (0.00 sec)
```

MySQL8.0默认会创建两个资源组，一个是`USR_default`另一个是`SYS_default`。

* PERFORMANCE_SCHEMA.THREADS

PERFORMANCE_SCHEMA库下的THREADS表中，可以查看当前线程使用资源组的情况:

```
mysql> mysql> select * from performance_schema.threads limit 5;
+-----------+------------------------------+------------+----------------+------------------+------------------+----------------+---------------------+------------------+-------------------+------------------+------------------+------+--------------+---------+-----------------+--------------+----------------+
| THREAD_ID | NAME                         | TYPE       | PROCESSLIST_ID | PROCESSLIST_USER | PROCESSLIST_HOST | PROCESSLIST_DB | PROCESSLIST_COMMAND | PROCESSLIST_TIME | PROCESSLIST_STATE | PROCESSLIST_INFO | PARENT_THREAD_ID | ROLE | INSTRUMENTED | HISTORY | CONNECTION_TYPE | THREAD_OS_ID | RESOURCE_GROUP |
+-----------+------------------------------+------------+----------------+------------------+------------------+----------------+---------------------+------------------+-------------------+------------------+------------------+------+--------------+---------+-----------------+--------------+----------------+
|         1 | thread/sql/main              | BACKGROUND |           NULL | NULL             | NULL             | mysql          | NULL                |            96053 | NULL              | NULL             |             NULL | NULL | YES          | YES     | NULL            |         9130 | SYS_default    |
|         3 | thread/innodb/io_ibuf_thread | BACKGROUND |           NULL | NULL             | NULL             | NULL           | NULL                |             NULL | NULL              | NULL             |             NULL | NULL | YES          | YES     | NULL            |         9135 | SYS_default    |
|         4 | thread/innodb/io_log_thread  | BACKGROUND |           NULL | NULL             | NULL             | NULL           | NULL                |             NULL | NULL              | NULL             |             NULL | NULL | YES          | YES     | NULL            |         9136 | SYS_default    |
|         5 | thread/innodb/io_read_thread | BACKGROUND |           NULL | NULL             | NULL             | NULL           | NULL                |             NULL | NULL              | NULL             |             NULL | NULL | YES          | YES     | NULL            |         9138 | SYS_default    |
|         6 | thread/innodb/io_read_thread | BACKGROUND |           NULL | NULL             | NULL             | NULL           | NULL                |             NULL | NULL              | NULL             |             NULL | NULL | YES          | YES     | NULL            |         9137 | SYS_default    |
+-----------+------------------------------+------------+----------------+------------------+------------------+----------------+---------------------+------------------+-------------------+------------------+------------------+------+--------------+---------+-----------------+--------------+----------------+
5 rows in set (0.00 sec)
```

其中`RESOURCE_GROUP`字段显示线程使用的是哪个资源组。

## 如何使用资源组
假设我们现在需要对跑批任务创建一个资源组。

### 创建一个资源组

```
mysql> CREATE RESOURCE GROUP Batch
    ->   TYPE = USER
    ->   VCPU = 2-3
    ->   THREAD_PRIORITY = 10;
Query OK, 0 rows affected (0.03 sec)

mysql> select * from information_schema.resource_groups;
+---------------------+---------------------+------------------------+----------+-----------------+
| RESOURCE_GROUP_NAME | RESOURCE_GROUP_TYPE | RESOURCE_GROUP_ENABLED | VCPU_IDS | THREAD_PRIORITY |
+---------------------+---------------------+------------------------+----------+-----------------+
| USR_default         | USER                |                      1 | 0-23     |               0 |
| SYS_default         | SYSTEM              |                      1 | 0-23     |               0 |
| Batch               | USER                |                      1 | 2-3      |              10 |
+---------------------+---------------------+------------------------+----------+-----------------+
3 rows in set (0.00 sec)
```

### 指定使用资源组
将创建的Batch资源组绑定到执行的线程上，有两种方式：

* 方式一
从PERFORMANCE_SCHEMA.THREADS表中查找需要绑定执行的线程ID(注意：THREADS表中的THREAD_ID和SHOW PROCESSLIST的ID不等同):

查看需要绑定需要的线程的THREAD_ID：

```
mysql> select * from performance_schema.threads where TYPE='FOREGROUND';
+-----------+--------------------------------+------------+----------------+------------------+------------------+----------------+---------------------+------------------+-------------------+------------------------------------------------------------------+------------------+------+--------------+---------+-----------------+--------------+----------------+
| THREAD_ID | NAME                           | TYPE       | PROCESSLIST_ID | PROCESSLIST_USER | PROCESSLIST_HOST | PROCESSLIST_DB | PROCESSLIST_COMMAND | PROCESSLIST_TIME | PROCESSLIST_STATE | PROCESSLIST_INFO                                                 | PARENT_THREAD_ID | ROLE | INSTRUMENTED | HISTORY | CONNECTION_TYPE | THREAD_OS_ID | RESOURCE_GROUP |
+-----------+--------------------------------+------------+----------------+------------------+------------------+----------------+---------------------+------------------+-------------------+------------------------------------------------------------------+------------------+------+--------------+---------+-----------------+--------------+----------------+
|        61 | thread/sql/compress_gtid_table | FOREGROUND |              5 | NULL             | NULL             | NULL           | Daemon              |            96613 | Suspending        | NULL                                                             |                1 | NULL | YES          | YES     | NULL            |         9211 | SYS_default    |
|        65 | thread/sql/one_connection      | FOREGROUND |             10 | root             | localhost        | NULL           | Query               |                0 | Sending data      | select * from performance_schema.threads where TYPE='FOREGROUND' |             NULL | NULL | YES          | YES     | Socket          |         9741 | USR_default    |
+-----------+--------------------------------+------------+----------------+------------------+------------------+----------------+---------------------+------------------+-------------------+------------------------------------------------------------------+------------------+------+--------------+---------+-----------------+--------------+----------------+
2 rows in set (0.00 sec)
```

绑定将线程与Batch资源组绑定：

```
SET RESOURCE GROUP Batch FOR 65;
```

查看绑定结果:

```
mysql> select THREAD_ID,NAME,TYPE,PROCESSLIST_ID,RESOURCE_GROUP from performance_schema.threads where TYPE='FOREGROUND';
+-----------+--------------------------------+------------+----------------+----------------+
| THREAD_ID | NAME                           | TYPE       | PROCESSLIST_ID | RESOURCE_GROUP |
+-----------+--------------------------------+------------+----------------+----------------+
|        61 | thread/sql/compress_gtid_table | FOREGROUND |              5 | SYS_default    |
|        65 | thread/sql/one_connection      | FOREGROUND |             10 | Batch          |
+-----------+--------------------------------+------------+----------------+----------------+
2 rows in set (0.00 sec)
```

* 方式二

采用Optimizer Hints的方式指定SQL使用的资源组：

```
SELECT /*+ RESOURCE_GROUP(Batch) */ * FROM t2 ;
```

### 修改资源组配置
可能跑批任务使用CPU资源不够，那就需要修改资源组的配置。

```
ALTER RESOURCE GROUP Batch VCPU = 10-20;
```

修改资源组优先级：

```
ALTER RESOURCE GROUP Batch THREAD_PRIORITY = 5;
```

禁止使用资源组：

```
ALTER RESOURCE GROUP Batch DISABLE FORCE;
```
### 删除资源组
对于不用的资源组可以删除

```
DROP RESOURCE GROUP Batch;
```

## 使用注意点

* CREATE RESOURCE GROUP、ALTER RESOURCE GROUP、DROP RESOURCE GROUP等SQL语句并不会被记录到binlog中，不会被复制到从库。
* 如果MySQL安装了thread pool插件，无法使用RESOURCE GROUP特性
* macOS 平台上不支持RESOURCE GROUP特性
* FreeBSD 和 Solaris 平台上无法指定资源组优先级(priorities),所有线程运行在0状态。
* Linux 平台上需要开启 CAP_SYS_NICE 特性才能使用RESOURCE GROUP

```
检查mysqld进程是否开启CAP_SYS_NICE特性
[root@qdata-sto3 /root]
#getcap /home/mysql/program/mysql8.0/bin/mysqld

给mysqld进程开启CAP_SYS_NICE特性
[root@qdata-sto3 /root]
#setcap cap_sys_nice+ep /home/mysql/program/mysql8.0/bin/mysqld

检查是否开启成功
[root@qdata-sto3 /root]
#getcap /home/mysql/program/mysql8.0/bin/mysqld
/home/mysql/program/mysql8.0/bin/mysqld = cap_sys_nice+ep
```

* Windows平台上线程优先级只有5个等级，分别为THREAD_PRIORITY_HIGHEST、THREAD_PRIORITY_ABOVE_NORMAL、THREAD_PRIORITY_NORMAL、THREAD_PRIORITY_BELOW_NORMAL、THREAD_PRIORITY_LOWEST


## 总结
Resource Group是MySQL在8.0中增加的资源管理的特性，从一个DBA的角度，可以更加灵活、便捷的管理数据库使用的资源，这是一个令人眼前一亮的特性。Oracle在10g的时候已经推出了Resource Manager的特性，MySQL的Resource Group目前还很简单只能控制CPU的资源，但是相信后续Resource Group能操控的资源类型会越来越多。期待MySQL8.0早日成为一个成熟稳定的版本。



> 博客地址：https://win-man.github.io/  
> 公众号：欢迎关注  

![](https://user-gold-cdn.xitu.io/2018/8/16/165435ce71d2b88b?w=258&h=258&f=jpeg&s=26568)
