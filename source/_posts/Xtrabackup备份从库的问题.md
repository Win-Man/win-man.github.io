---
title: Xtrabackup备份从库的问题
date: 2017-07-11 08:14:05
tags:
- Xtrabackup
- 备份
categories:
- MySQL
---
## 环境介绍
* 数据库信息
主库IP：192.168.1.130    
从库IP：192.168.1.132
主库MySQL版本：5.6.36
从库MySQL版本：5.6.35
* 配置信息
主从开启GTID复制
主库参数配置:gtid-mode=ON;enforce-gtid-consistency=ON
从库参数配置:gtid-mode=ON,enforce-gtid-consistency=ON;log-slave-update=ON;master_info_repository=TABLE;relay_log_info_repository=TABLE

## 测试步骤

* 搭建主从复制，搭建步骤在此省略
* 使用Xtrabackup备份从库数据
```shell
备份从库
[root@host-192-168-1-132 backup]# innobackupex --defaults-file=/etc/my.cnf --user=xtrabackup --password=xtrabackup --slave-info --no-timestamp /tmp/backup
应用日志
[root@host-192-168-1-132 backup]# innobackupex --defaults-file=/etc/my.cnf --apply-log /tmp/backup
```
* 清空从库数据，并使用备份恢复从库
```shell
[root@host-192-168-1-132 mysqldata1]# innobackupex --defaults-file=/etc/my.cnf --copy-back /tmp/backup/
[root@host-192-168-1-132 mysqldata1]# chown -R mysql:mysql /home/mysql
```
* 查看xtrabckup中备份的slave信息
```shell
[root@host-192-168-1-132 backup]# cat xtrabackup_slave_info 
SET GLOBAL gtid_purged='05dff9fd-5c97-11e7-b3f4-fa163ea44438:1-7';
CHANGE MASTER TO MASTER_AUTO_POSITION=1
```
* 启动从库，查看从库信息
```shell
[root@host-192-168-1-132 mysqldata1]# service mysqld start

mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: 
                  Master_Host: 192.168.1.130
                  Master_User: repl
                  Master_Port: 6306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000002
          Read_Master_Log_Pos: 1615
               Relay_Log_File: mysql-relay-bin.000002
                Relay_Log_Pos: 649
        Relay_Master_Log_File: mysql-bin.000002
             Slave_IO_Running: No
            Slave_SQL_Running: No
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 1856
              Relay_Log_Space: 0
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: NULL
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 0
                  Master_UUID: 05dff9fd-5c97-11e7-b3f4-fa163ea44438
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: 
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 1
1 row in set (0.00 sec)

mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      151 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)

mysql> set global gtid_purged='05dff9fd-5c97-11e7-b3f4-fa163ea44438:1-7';
Query OK, 0 rows affected (0.08 sec)

mysql> start slave;
ERROR 1872 (HY000): Slave failed to initialize relay log info structure from the repository
```
* start slave的时候出现错误，于是重置了slave信息，重新change master 解决问题
```shell
mysql> reset slave all;
Query OK, 0 rows affected (0.03 sec)

mysql> change master to
    -> master_host='192.168.1.130',
    -> master_user='repl',
    -> master_password='repl',
    -> master_port=6306,
    -> master_auto_position=1;
Query OK, 0 rows affected, 2 warnings (0.35 sec)

mysql> start slave;
Query OK, 0 rows affected (0.02 sec)

mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.1.130
                  Master_User: repl
                  Master_Port: 6306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000002
          Read_Master_Log_Pos: 1856
               Relay_Log_File: mysql-relay-bin.000002
                Relay_Log_Pos: 408
        Relay_Master_Log_File: mysql-bin.000002
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 1856
              Relay_Log_Space: 612
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 3656
                  Master_UUID: 05dff9fd-5c97-11e7-b3f4-fa163ea44438
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for the slave I/O thread to update it
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 05dff9fd-5c97-11e7-b3f4-fa163ea44438:1-7
                Auto_Position: 1
1 row in set (0.00 sec)
```

## 总结
* XtraBackup备份从库的时候，利用备份恢复数据之后，第一次启动数据库的时候，不会启动I/O线程和SQL线程
* Xtrabackup备份从库，虽然保存了原先slave中chang master to的信息，但是不能直接start slave；需要reset slave all之后重新指定主库信息
* Xtrabackup备份从库，不会备份GTID_PURGED的信息，需要恢复的时候，自己指定
