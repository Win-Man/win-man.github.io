---
title: MySQL 主从环境配置
date: 2016-11-07 20:46:25
tags:
- 复制
- MySQL
categories:
- MySQL
---

# 1.前言
　　
# 2.环境

>操作系统： CentOS6.5
>MySQL：mysql-5.6.34

# 3.二进制包方式安装 MySQL

## 3.1 卸载 Linux 自带的 MySQL

Linux 系统自带了 MySQL 数据库，但是对于我们来说版本低了点，所以我们还是自己重新安装一遍 MySQL，为防止冲突，我们需要将原先版本的 MySQL 卸载。

1. 查看 MySQL 及相关的 RPM 包

```c
rpm -qa | grep -i mysql
```

![](http://oc4wmeyj8.bkt.clouddn.com/%E6%9F%A5%E8%AF%A2%E8%87%AA%E5%B8%A6mysql.PNG)

2. 卸载 MySQL

```c
yum -y remove mysql-5.1.71-1.el6
```

3. 查看是否卸载

![](http://oc4wmeyj8.bkt.clouddn.com/%E5%8D%B8%E8%BD%BDmysql%E5%90%8E.PNG)

## 3.2 安装 MySQL

1. 下载 MySQL

通过 `wget` 命令下载 MySQL 的 tar.gz 文件

2. 解压 tar.gz 文件

```c
cd /usr/local
tar zxvf {下载文件所在路径}.tar.gz
```

3. 重名文件夹

```c
mv mysql-5.6.34-linux-glibc2.5-i686/ mysql
```

4. 更改 mysql 文件夹属性

```c
cd mysql/
groupadd mysql
useradd -g mysql mysql
chown -R mysql .
chgrp -R mysql .
``` 

5. 编译安装 MySQL

```c
scripts/mysql_install_db --user=mysql --basedir=/usr/local/mysql/ --datadir=/usr/local/mysql/data/
```

6. 配置 my.cnf 文件

从 mysql 安装目录下拷贝一份 my.cnf 文件到 /etc/ 目录下，如果原先 /etc/ 存在 my.cnf 文件，则覆盖。

在 `[mysqld]`中添加：

```c
basedir = /usr/local/mysql
datadir = /usr/local/mysql/data
port = 3306
server_id = 1
```

7. 启动 MySQL 服务

```c
cp support-files/mysql.server /etc/init.d/mysqld
ln -s /usr/local/mysql/bin/mysql /usr/bin
service mysqld start
```

8. 验证 MySQL 服务是否开启

```c
ps -ef | grep mysqld
```


# 4.配置主从环境

## 4.1 准备两台 Linux 环境服务器

分别在两台服务器上按上述步骤安装 MySQL。

## 4.2 配置主库

1. 修改主库所在服务器上的 my.cnf 文件

```c
vi /etc/my.cnf
```

在 my.cnf 文件中添加

```c
log_bin = master_mysql_bin
```

该参数配置表示开启 MySQL 的二进制日志。

2. 重启 MySQL 服务

```c
service mysqld restart
```

3. 查看 master 节点状态

```c
mysql> show master status;
```

![](http://oc4wmeyj8.bkt.clouddn.com/master_status.PNG)

记录下红色标记处的信息，配置 slave 节点时需要。

4. 主库中增加复制账号

```c
mysql> grant replication slave on *.* to 'repl'@'192.168.222.%' identified by 'replsafe';
```



## 4.3 配置从库

1. 修改从库所在服务器上的 my.cnf 文件

```c
vi /etc/my.cnf
```

在 my.cnf 文件中添加

```c
log_bin = slave_mysql_bin
relay_log = mysql_relay_bin
```

修改从库上的二进制日志和中继日志的命名规则

2. 重启 MySQL 服务

```c
service mysqld restart
```

3. 连接到主库

```c
mysql> change master to
    -> master_host='192.168.222.131',
    -> master_port=3306,
    -> master_user='repl',
    -> master_password='replsafe',
    -> master_log_file='master_mysql_bin.000001',
    -> master_log_pos=120;
Query OK, 0 rows affected, 2 warnings (0.11 sec)
```

4. 开启复制

```c
start slave;
```

5. 查看从库复制状态

```c
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.222.131
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: master_mysql_bin.000001
          Read_Master_Log_Pos: 330
               Relay_Log_File: mysql_relay_bin.000002
                Relay_Log_Pos: 500
        Relay_Master_Log_File: master_mysql_bin.000001
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
          Exec_Master_Log_Pos: 330
              Relay_Log_Space: 673
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
             Master_Server_Id: 131
                  Master_UUID: 249f7c7a-9bc8-11e6-8aaf-000c29b971bb
             Master_Info_File: /usr/local/mysql/data/master.info
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
            Executed_Gtid_Set: 
                Auto_Position: 0
1 row in set (0.00 sec)
```

通过 `show slave status\G` 显示出来的内容如上，则表示主备环境搭建成功。

6. 验证复制功能

在主库上创建一个数据库，创建一张表，并想表中插入数据，查看从库能否有相同数据。

```c
mysql> create database master_slave;
Query OK, 1 row affected (0.00 sec)

mysql> use master_slave;
Database changed
mysql> create table tb1(id int);
Query OK, 0 rows affected (0.14 sec)

mysql> insert into tb1 values(1);
Query OK, 1 row affected (0.03 sec)

mysql> select * from tb1;
+------+
| id   |
+------+
|    1 |
+------+
1 row in set (0.00 sec)

```

查看从库上是否有相同数据

```c
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| master_slave       |
| mysql              |
| performance_schema |
| test               |
+--------------------+
5 rows in set (0.00 sec)

mysql> use master_slave;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+------------------------+
| Tables_in_master_slave |
+------------------------+
| tb1                    |
+------------------------+
1 row in set (0.00 sec)

mysql> select * from tb1;
+------+
| id   |
+------+
|    1 |
+------+
1 row in set (0.00 sec)
```

验证复制功能成功。

# 5.可能遇到的问题

**问题一：**
在启动复制功能时，查看 slave 节点状态发现：

![](http://oc4wmeyj8.bkt.clouddn.com/2003%E9%94%99%E8%AF%AF.PNG)

**原因：**主库服务器上防火墙开启导致，从库连接不上主库

**解决方案：**在主库服务器上关闭防火墙 `service iptables stop`


> 博客地址：https://win-man.github.io/  
> 公众号：欢迎关注  

![](https://user-gold-cdn.xitu.io/2018/8/16/165435ce71d2b88b?w=258&h=258&f=jpeg&s=26568)
