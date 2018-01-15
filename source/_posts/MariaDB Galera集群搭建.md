---
title: MariaDB Galera集群搭建
date: 2018-01-15 11:08:13
tags:
- MariaDB
- Galera
categories:
- MariaDB
---

# 1、概述
## 1.1、目的

## 1.2、环境信息
* 操作系统版本CentOS Linux release 7.2.1511 (Core)
* MariaDB:10.1.14-MariaDB MariaDB Server
* 物理机1：node1：10.10.90.167
* 物理机2：node2：10.10.90.174
* 物理机3：node3：10.10.90.181

# 2、Galera集群搭建步骤
## 2.1、MariaDB安装步骤
### MariaDB参数
MariaDB-10.1.14安装步骤与MySQL-5.6一致，只是一些在MySQL中的参数，在MarIaDB中不能使用或者被忽略。在QData的参数模板中，以下参数会导致数据库初始化话**失败**：
```
gtid-mode=on # GTID only
enforce-gtid-consistency=true # GTID only
slave_parallel_type=LOGICAL_CLOCK
innodb_undo_log_truncate=ON
log_timestamps=SYSTEM
slave_preserve_commit_order=ON
```
以下参数会使数据库初始化的过程中出现警告:
```
slave_pending_jobs_size_max=128M
table_open_cache = 4096
binlog-rows-query-log-events
master-info-repository
relay-log-info-repository
slave-parallel-workers
```
使用的参数模板如下:
```
[client]
loose_default-character-set = utf8
port=3306
socket=/home/mysql/data/mysqldata1/sock/mysql.sock
user=admin

[mysqldump]
quick
max_allowed_packet = 2G
default-character-set = utf8

[mysql]
no-auto-rehash
show-warnings
prompt="\\u@\\h : \\d \\r:\\m:\\s> "
default-character-set = utf8

[myisamchk]
key_buffer = 512M
sort_buffer_size = 512M
read_buffer = 8M
write_buffer = 8M

[mysqlhotcopy]
interactive-timeout

[mysqld_safe]
user=mysql
open-files-limit = 65535

[mysqld]
#large-pages
#***********************************common parameters******************************
read_only=on
#memlock
default-storage-engine = INNODB
character-set-server=utf8
collation_server = utf8_bin

user=mysql
port=3306
socket=/home/mysql/data/mysqldata1/sock/mysql.sock
pid-file=/home/mysql/data/mysqldata1/sock/mysql.pid
datadir=/home/mysql/data/mysqldata1/mydata
tmpdir=/home/mysql/data/mysqldata1/tmpdir

skip-name-resolve
skip_external_locking

lower_case_table_names=1
event_scheduler=0
back_log=512
default-time-zone='+8:00'

max_connections = 3000
max_connect_errors=99999
max_allowed_packet = 64M
#slave_pending_jobs_size_max=128M
max_heap_table_size = 8M
max_length_for_sort_data = 16k

wait_timeout=172800
interactive_timeout=172800

net_buffer_length = 8K
read_buffer_size = 2M
read_rnd_buffer_size = 2M
sort_buffer_size = 2M
join_buffer_size = 4M
binlog_cache_size = 2M

#table_open_cache = 4096
table_open_cache_instances = 2
table_definition_cache = 4096
thread_cache_size = 512
tmp_table_size = 8M

query_cache_size=0
query_cache_type=OFF
explicit_defaults_for_timestamp=ON

#*******************************  Logs related settings ***************************
log-error=/home/mysql/data/mysqldata1/log/error.log
long_query_time = 1
slow_query_log
slow_query_log_file=/home/mysql/data/mysqldata1/slowlog/slow-query.log
log_slow_slave_statements
#log_queries_not_using_indexes

#*******************************  Replication related settings **********************

#### For Master
server-id=3306191
log-bin=/home/mysql/data/mysqldata1/binlog/mysql-bin
binlog-format=ROW
binlog-checksum=CRC32
#binlog-rows-query-log-events=1
max_binlog_size = 512M
expire_logs_days=15
sync_binlog=1
master-verify-checksum=1
#master-info-repository=TABLE
auto_increment_increment=2
auto_increment_offset=2

#### For Slave
relay-log=/home/mysql/data/mysqldata1/relaylog/mysql-relay-bin
#relay-log-info-repository=TABLE
relay-log-recovery=1
#slave-skip-errors=1022,1032,1062,1236
slave-sql-verify-checksum=1
log_bin_trust_function_creators=1
log_slave_updates=1
slave-net-timeout=10
skip-slave-start

#******************************* MyISAM Specific options ****************************
key_buffer_size = 8M
bulk_insert_buffer_size = 8M
myisam_sort_buffer_size = 64M
myisam_max_sort_file_size = 10G
myisam_repair_threads = 1
myisam_recover_options=force

# ***************************** INNODB Specific options ****************************
#### Data options
#innodb_data_home_dir = /home/mysql/data/mysqldata1/innodb_ts
innodb_data_file_path = ibdata1:2048M:autoextend
innodb_file_per_table
innodb_file_format = barracuda
innodb_file_format_max = barracuda
innodb_file_format_check = ON
innodb_strict_mode = 1
innodb_flush_method = O_DIRECT
innodb_checksum_algorithm=crc32
innodb_autoinc_lock_mode=2

#### Buffer Pool options
innodb_buffer_pool_size = 4G
innodb_buffer_pool_instances = 2
innodb_max_dirty_pages_pct = 75
innodb_adaptive_flushing = ON
innodb_flush_neighbors = 0
innodb_lru_scan_depth = 4096
#innodb_change_buffering = inserts
innodb_old_blocks_time = 1000

#### Redo options
innodb_log_group_home_dir = /home/mysql/data/mysqldata1/innodb_log
innodb_log_buffer_size = 64M
innodb_log_file_size = 2G
innodb_log_files_in_group = 2
innodb_flush_log_at_trx_commit = 1
innodb_fast_shutdown = 2
innodb_support_xa = ON

#### Transaction options
innodb_thread_concurrency = 64
innodb_lock_wait_timeout = 120
innodb_rollback_on_timeout = 1
transaction_isolation = READ-COMMITTED

#### IO options
innodb_read_io_threads = 8
innodb_write_io_threads = 16
innodb_io_capacity = 20000
innodb_use_native_aio = 1

#### Undo options
innodb_undo_directory = /home/mysql/data/mysqldata1/undo/
innodb_undo_tablespaces=16
innodb_purge_threads = 4
innodb_purge_batch_size = 512
innodb_max_purge_lag = 65536

#### 5.6/5.7 GTID
#gtid-mode=on # GTID only
#enforce-gtid-consistency=true # GTID only

#### mysql 5.7
#super_read_only=on
secure_file_priv=/data2/
#slave_parallel_workers=16
#slave_parallel_type=LOGICAL_CLOCK
#innodb_undo_log_truncate=ON
#log_timestamps=SYSTEM
#slave_preserve_commit_order=ON

#### mariadb
gtid_strict_mode=on
gtid_domain_id=191

#### galera
wsrep_on=on
wsrep_provider=/usr/local/mysql/lib/galera/libgalera_smm.so
wsrep_provider_options="gcache.size=1G"

wsrep_cluster_name=wqglr_0001
wsrep_cluster_address=gcomm://10.10.90.167:4567,10.10.90.174:4567,10.10.90.181:4567
wsrep_node_name = node3
wsrep_node_address=10.10.90.181:4567
#
#wsrep_sst_method=xtrabackup-v2
#wsrep_sst_auth="xtrabackup:xtrabackuppass"
#wsrep_slave_threads=16
#
#innodb_support_xa = OFF
#sync_binlog=0
#innodb_flush_log_at_trx_commit = 0

```
### 安装步骤
*  创建目录
```
mkdir -p /data2/mysqldata1/{binlog,conf,innodb_log,innodb_ts,log,mydata,relaylog,slowlog,sock,tmpdir,undo}
mkdir -p /home/mysql/{data,program}
ln -s /data2/mysqldata1 /home/mysql/data/mysqldata1
```

* 下载二进制安装包，并上传到对应位置
* 将安装包解压至指定目录,并建立软链至/usr/local
```
tar -zxvf mariadb-10.1.14-linux-x86_64 -c /home/mysql/program/
ln -s /home/mysql/program/mariadb-10.1.14-linux-x86_64 /usr/local/mysql
```
* 添加用户以及用户组
```
groupadd mysql
useradd -r -g mysql mysql
```
* 更改目录权限
```
chown -R mysql:mysql /data/mysqldata1
```
* 填写配置文件
将上面的配置文件内容写到/etc/my.cnf中
* 初始化数据库
```
cd /usr/local/mysql
./scripts/mysql_install_db --defaults-file=/etc/my.cnf --user=mysql
```
**验证安装成功：**
查看是否有两个OK
* 添加service启动方式
```
cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysqld
```
* 尝试启动数据库，验证数据库正常
```
service mysqld start
```

** 按照相同的步骤安装三个节点**
## 2.2、Galera集群安装步骤
### 2.2.1、第一个节点安装
安装步骤:
* 在配置文件中填写galera集群的配置参数
一下为最基础的参数配置，一些调优等的配置参数不列在这边
```
gtid_strict_mode=on
gtid_domain_id=191
wsrep_on=on
wsrep_provider=/usr/local/mysql/lib/galera/libgalera_smm.so
wsrep_provider_options="gcache.size=1G"
wsrep_cluster_name=wqglr_0001
wsrep_cluster_address=gcomm://10.10.90.167:4567,10.10.90.174:4567,10.10.90.181:4567
wsrep_node_name = node3
wsrep_node_address=10.10.90.181:4567
```
* 启动第一个节点
galera集群启动第一个节点的时候需要加上，--wsrep-new-cluster来初始化一个集群，这个只需要在**第一个节点第一次启动**的时候添加，之后该节点启动时不需要添加该参数，因为这个参数的表示初始化一个新的集群，每次添加该参数，都会初始化一个新的集群
```
cd /usr/local/mysql/bin
./mysqld_safe --defaults-file=/etc/my.cnf --wsrep-new-cluster &
```
* 进入数据库检查，集群状态 show status like '%wsrep_%'
```
root@localhost : (none) 11:00:08> show status like '%wsrep_%';
+------------------------------+----------------------------------------------------------------+
| Variable_name                | Value                                                          |
+------------------------------+----------------------------------------------------------------+
| wsrep_apply_oooe            | 0.000000                                                      |
| wsrep_apply_oool            | 0.000000                                                      |
| wsrep_apply_window          | 1.000000                                                      |
| wsrep_causal_reads          | 0                                                              |
| wsrep_cert_deps_distance    | 1.000000                                                      |
| wsrep_cert_index_size        | 1                                                              |
| wsrep_cert_interval          | 0.000000                                                      |
| wsrep_cluster_conf_id        | 10                                                            |
| wsrep_cluster_size          | 3                                                              |
| wsrep_cluster_state_uuid    | 5ace7bde-f5d3-11e7-81fd-b7f62639a9a0                          |
| wsrep_cluster_status        | Primary                                                        |
| wsrep_commit_oooe            | 0.000000                                                      |
| wsrep_commit_oool            | 0.000000                                                      |
| wsrep_commit_window          | 1.000000                                                      |
| wsrep_connected              | ON                                                            |
| wsrep_evs_delayed            | 829f35c1-f5d6-11e7-9525-b21c1d0f3f09:tcp://10.10.90.191:4567:1 |
| wsrep_evs_evict_list        |                                                                |
| wsrep_evs_repl_latency      | 0/0/0/0/0                                                      |
| wsrep_evs_state              | OPERATIONAL                                                    |
| wsrep_flow_control_paused    | 0.000000                                                      |
| wsrep_flow_control_paused_ns | 0                                                              |
| wsrep_flow_control_recv      | 0                                                              |
| wsrep_flow_control_sent      | 0                                                              |
| wsrep_gcomm_uuid            | 5acdf3cc-f5d3-11e7-bcb5-46c7c07b299e                          |
| wsrep_incoming_addresses    | 10.10.90.181:3306,10.10.90.167:3306,10.10.90.174:3306          |
| wsrep_last_committed        | 3                                                              |
| wsrep_local_bf_aborts        | 0                                                              |
| wsrep_local_cached_downto    | 1                                                              |
| wsrep_local_cert_failures    | 0                                                              |
| wsrep_local_commits          | 0                                                              |
| wsrep_local_index            | 1                                                              |
| wsrep_local_recv_queue      | 0                                                              |
| wsrep_local_recv_queue_avg  | 0.074074                                                      |
| wsrep_local_recv_queue_max  | 2                                                              |
| wsrep_local_recv_queue_min  | 0                                                              |
| wsrep_local_replays          | 0                                                              |
| wsrep_local_send_queue      | 0                                                              |
| wsrep_local_send_queue_avg  | 0.000000                                                      |
| wsrep_local_send_queue_max  | 1                                                              |
| wsrep_local_send_queue_min  | 0                                                              |
| wsrep_local_state            | 4                                                              |
| wsrep_local_state_comment    | Synced                                                        |
| wsrep_local_state_uuid      | 5ace7bde-f5d3-11e7-81fd-b7f62639a9a0                          |
| wsrep_protocol_version      | 7                                                              |
| wsrep_provider_name          | Galera                                                        |
| wsrep_provider_vendor        | Codership Oy <info@codership.com>                              |
| wsrep_provider_version      | 25.3.15(r3578)                                                |
| wsrep_ready                  | ON                                                            |
| wsrep_received              | 27                                                            |
| wsrep_received_bytes        | 3661                                                          |
| wsrep_repl_data_bytes        | 356                                                            |
| wsrep_repl_keys              | 1                                                              |
| wsrep_repl_keys_bytes        | 31                                                            |
| wsrep_repl_other_bytes      | 0                                                              |
| wsrep_replicated            | 1                                                              |
| wsrep_replicated_bytes      | 451                                                            |
| wsrep_thread_count          | 2                                                              |
+------------------------------+----------------------------------------------------------------+
57 rows in set (0.00 sec)
```
重点关注几个状态:
`wsrep_connected=on` 表示链接已开启
`wsrep_local_index=1` 表示集群中的索引值
`wsrep_cluster_size=3` 表示集群中节点的数量
`wsrep_incoming_addresses =0.10.90.181:3306,10.10.90.167:3306,10.10.90.174:3306`集群中节点的访问地址


### 2.2.2、加入一个数据节点
安装步骤
* 在配置文件中填写galera集群的配置参数
一下为最基础的参数配置，一些调优等的配置参数不列在这边
```
gtid_strict_mode=on
gtid_domain_id=191
wsrep_on=on
wsrep_provider=/usr/local/mysql/lib/galera/libgalera_smm.so
wsrep_provider_options="gcache.size=1G"
wsrep_cluster_name=wqglr_0001
wsrep_cluster_address=gcomm://10.10.90.167:4567,10.10.90.174:4567,10.10.90.181:4567
wsrep_node_name = node3
wsrep_node_address=10.10.90.181:4567
```
* 启动节点
```
service mysqld start
```
* 进入数据库检查，集群状态 show status like '%wsrep_%'

### 2.2.3、加入一个仲裁节点
仲裁节点不需要初始化数据库，不需要进行配置，但是数据目录需要创建，数据库程序需要部署
* 启动仲裁节点
```
cd /usr/local/mysql/bin
./garbd --address="gcomm://10.10.90.167:4567,10.10.90.174:4567,10.10.90.181:4567" --group=wqglr_0001 --name='10.10.90.181:4567' --log=/home/mysql/data/mysqldata1/log/garbd.log -d
```
* 检查仲裁节点加入情况
node1或者node2上show status like '%wsrep_cluster%';查看集群节点数量，发现集群节点数量增加1
```
root@localhost : (none) 01:23:30> show status like '%wsrep_cluster%';
+--------------------------+--------------------------------------+
| Variable_name            | Value                                |
+--------------------------+--------------------------------------+
| wsrep_cluster_conf_id    | 3                                    |
| wsrep_cluster_size      | 3                                    |
| wsrep_cluster_state_uuid | 75b1d970-f681-11e7-a221-af292503fa6e |
| wsrep_cluster_status    | Primary                              |
+--------------------------+--------------------------------------+
4 rows in set (0.00 sec)
```

### 2.2.4、验证数据同步
在node1上：
```
create database test1;
use test1;
create table test1(id int primary key auto_increment,name varchar(20));
insert  into test1(`name`) values('test1');
```
在node2上
```
create database test2;
use test2;
create table test1(id int primary key auto_increment,name varchar(20));
insert  into test1(`name`) values('test1');
```
在两个节点上查询数据，发现数据复制正常

### 2.2.5、脑裂验证





# 3、安装过程中的问题
## galera集群启动过程中，报libssl和libcrypto库缺失
错误日志：
```
2018-01-09 16:19:28 140122035881856 [Warning] 'table-open-cache-instances' is MySQL 5.6 compatible option. Not used or needed in MariaDB.
2018-01-09 16:19:28 140122035881856 [Note] /usr/local/mysql/bin/mysqld (mysqld 10.1.14-MariaDB) starting as process 31625 ...
2018-01-09 16:19:28 140122035881856 [Note] WSREP: Read nil XID from storage engines, skipping position init
2018-01-09 16:19:28 140122035881856 [Note] WSREP: wsrep_load(): loading provider library '/usr/local/mysql/lib/galera/libgalera_smm.so'
2018-01-09 16:19:28 140122035881856 [ERROR] WSREP: wsrep_load(): dlopen(): libssl.so.6: cannot open shared object file: No such file or directory
2018-01-09 16:19:28 140122035881856 [ERROR] WSREP: wsrep_load(/usr/local/mysql/lib/galera/libgalera_smm.so) failed: Invalid argument (22). Reverting to no provider.
2018-01-09 16:19:28 140122035881856 [Note] WSREP: Read nil XID from storage engines, skipping position init
2018-01-09 16:19:28 140122035881856 [Note] WSREP: wsrep_load(): loading provider library 'none'
2018-01-09 16:19:28 140122035881856 [ERROR] Aborting
```
解决方案：
```
#ldd libgalera_smm.so
        linux-vdso.so.1 =>  (0x00007ffda1bdc000)
        libpthread.so.0 => /lib64/libpthread.so.0 (0x00007fabbb35f000)
        librt.so.1 => /lib64/librt.so.1 (0x00007fabbb157000)
        libssl.so.6 => not found
        libcrypto.so.6 => not found
        libstdc++.so.6 => /lib64/libstdc++.so.6 (0x00007fabbae4e000)
        libm.so.6 => /lib64/libm.so.6 (0x00007fabbab4b000)
        libgcc_s.so.1 => /lib64/libgcc_s.so.1 (0x00007fabba935000)
        libc.so.6 => /lib64/libc.so.6 (0x00007fabba574000)
        /lib64/ld-linux-x86-64.so.2 (0x00007fabbbade000)


[root@node2 /lib64]
#cd /usr/lib64

[root@node2 /usr/lib64]
#ll libssl*
-rwxr-xr-x. 1 root 272536 Nov 20  2015 libssl3.so
lrwxrwxrwx. 1 root    16 Dec 27 15:31 libssl.so -> libssl.so.1.0.1e
lrwxrwxrwx. 1 root    16 Dec 27 15:30 libssl.so.10 -> libssl.so.1.0.1e
-rwxr-xr-x. 1 root 449864 Jun 29  2015 libssl.so.1.0.1e

[root@node2 /usr/lib64]
#ln -s libssl.so.1.0.1e libssl.so.6

[root@node2 /usr/lib64]
#ll libcrypto*
lrwxrwxrwx. 1 root      19 Dec 27 15:31 libcrypto.so -> libcrypto.so.1.0.1e
lrwxrwxrwx. 1 root      19 Dec 27 15:30 libcrypto.so.10 -> libcrypto.so.1.0.1e
-rwxr-xr-x. 1 root 2012880 Jun 29  2015 libcrypto.so.1.0.1e

[root@node2 /usr/lib64]
#ln -s libcrypto.so.1.0.1e libcrypto.so.6


#ldd /usr/local/mysql/lib/galera/libgalera_smm.so
        linux-vdso.so.1 =>  (0x00007ffecdbad000)
        libpthread.so.0 => /lib64/libpthread.so.0 (0x00007fe44169e000)
        librt.so.1 => /lib64/librt.so.1 (0x00007fe441496000)
        libssl.so.6 => /lib64/libssl.so.6 (0x00007fe441228000)
        libcrypto.so.6 => /lib64/libcrypto.so.6 (0x00007fe440e41000)
        libstdc++.so.6 => /lib64/libstdc++.so.6 (0x00007fe440b39000)
        libm.so.6 => /lib64/libm.so.6 (0x00007fe440836000)
        libgcc_s.so.1 => /lib64/libgcc_s.so.1 (0x00007fe440620000)
        libc.so.6 => /lib64/libc.so.6 (0x00007fe44025f000)
        /lib64/ld-linux-x86-64.so.2 (0x00007fe441e1d000)
        libgssapi_krb5.so.2 => /lib64/libgssapi_krb5.so.2 (0x00007fe440012000)
        libkrb5.so.3 => /lib64/libkrb5.so.3 (0x00007fe43fd2d000)
        libcom_err.so.2 => /lib64/libcom_err.so.2 (0x00007fe43fb29000)
        libk5crypto.so.3 => /lib64/libk5crypto.so.3 (0x00007fe43f8f6000)
        libdl.so.2 => /lib64/libdl.so.2 (0x00007fe43f6f2000)
        libz.so.1 => /lib64/libz.so.1 (0x00007fe43f4dc000)
        libkrb5support.so.0 => /lib64/libkrb5support.so.0 (0x00007fe43f2cc000)
        libkeyutils.so.1 => /lib64/libkeyutils.so.1 (0x00007fe43f0c8000)
        libresolv.so.2 => /lib64/libresolv.so.2 (0x00007fe43eeae000)
        libselinux.so.1 => /lib64/libselinux.so.1 (0x00007fe43ec88000)
        libpcre.so.1 => /lib64/libpcre.so.1 (0x00007fe43ea27000)
        liblzma.so.5 => /lib64/liblzma.so.5 (0x00007fe43e802000)

```

## 配置的innodb_data_home_dir和innodb_log_home_dir，导致rsync模式的sst传输报错
因为我们的galera集群的配置参数中没有指定：
```
#wsrep_sst_method=xtrabackup-v2
#wsrep_sst_auth="xtrabackup:xtrabackuppass"
```
所以galeta集群在进行sst的时候是通过默认的的rsync方式同步的，但是在这种方式下，不能自定义innodb_data_home_dir和innodb_log_home_dir,这是MariaDB Galera的BUG，如果要自定义这两个参数，可以选择xtrabackup的方式。
wsrep_sst_method可以指定为mysqldump,xtrabackup,rsync三种，xtrabackup的方式锁表时间最短
