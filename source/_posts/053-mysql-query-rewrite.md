---
title: MySQL query rewrite
date: 2021-02-23 10:52:34
tags:
- MySQL
categories:
- MySQL
---

# MySQL query rewrite

## 介绍

MySQL 从 5.7.6 版本开始支持 SQL 改写的功能，对于符合条件的 SQL 可以进行对应的修改。在 8.0.12 之前的版本只支持 SELECT 语句的改写，8.0.12 版本开始支持 SELECT/INSERT/REPLACE/UPDATE/DELETE 语句的改写。

## Query Rewrite Plugin 安装

安装 Query Rewrite Plugin 直接通过运行 install_rewriter.sql 中的 SQL 来进行安装即可，如果需要卸载的话，执行 uninstall_rewriter.sql 就可以，这两个 sql 文本文件存放在安装目录的 share 目录下。

- 执行 install_rewriter.sql 安装插件

```sql
# mysql -uroot -p -h127.0.0.1 < install_rewriter.sql
```

- 通过变量确认插件开启

```sql
root@127.0.0.1 : (none) 03:53:58> SHOW GLOBAL VARIABLES LIKE 'rewriter_enabled';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| rewriter_enabled | ON    |
+------------------+-------+
1 row in set (0.00 sec)
```

- 如果需要每次重启 rewriter_enabled 参数都是开启的话，可以在配置文件中配置对应参数

```sql
[mysqld]
rewriter_enabled=ON
```

- 动态修改参数可以通过以下方式开启或者关闭插件，在刚安装完插件的时候，默认是开启的

```sql
SET GLOBAL rewriter_enabled = ON;
SET GLOBAL rewriter_enabled = OFF;
```

安装完 Query Rewrite Plugin 插件之后自从创建一个 query_rewrite 的 database，该 database 下有一张 rewrite_rules 表，用于记录对应的改写规则。

```sql
root@127.0.0.1 : query_rewrite 06:48:28> show create table query_rewrite.rewrite_rules\G
*************************** 1. row ***************************
       Table: rewrite_rules
Create Table: CREATE TABLE `rewrite_rules` (
  `id` int NOT NULL AUTO_INCREMENT,
  `pattern` varchar(5000) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin NOT NULL,
  `pattern_database` varchar(20) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin DEFAULT NULL,
  `replacement` varchar(5000) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin NOT NULL,
  `enabled` enum('YES','NO') CHARACTER SET utf8mb4 COLLATE utf8mb4_bin NOT NULL DEFAULT 'YES',
  `message` varchar(1000) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin DEFAULT NULL,
  `pattern_digest` varchar(64) DEFAULT NULL,
  `normalized_pattern` varchar(100) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
1 row in set (0.00 sec)
```

## Query Rewrite Plugin 使用

使用语句改写的话，只需要将改写规则写入到 rewrite_rules 表中并且通过 CALL query_rewrite.flush_rewrite_rules() 加载生效即可。

比如将 SELECT ? 语句改写为 SELECT ? + 1 

```sql
root@127.0.0.1 : query_rewrite 07:32:41> INSERT INTO query_rewrite.rewrite_rules (pattern, replacement)
    -> VALUES('SELECT ?', 'SELECT ? + 1');
Query OK, 1 row affected (0.01 sec)

root@127.0.0.1 : query_rewrite 07:32:51> CALL query_rewrite.flush_rewrite_rules();
Query OK, 1 row affected (0.01 sec)

root@127.0.0.1 : query_rewrite 07:33:01> select 2;
+-------+
| 2 + 1 |
+-------+
|     3 |
+-------+
1 row in set, 1 warning (0.00 sec)

Note (Code 1105): Query 'select 2' rewritten to 'SELECT 2 + 1' by a query rewrite plugin
```

SQL 改写的话没有语句类型的限制，改写前的语句类型可以跟改写后的语句类型不一致，比如可以将 SELECT 语句改写为 INSERT 语句。如果调用 CALL query_rewrite.flush_rewrite_rules(); 使改写规则生效的时候报错了，可以查看 rewrite_rules 表中对应的 message 字段，会有具体的错误信息提示。

```sql
root@127.0.0.1 : query_rewrite 07:34:51> use gangshen;
Database changed
root@127.0.0.1 : gangshen 07:37:13> create table t(id int primary key,name varchar(20));
Query OK, 0 rows affected (0.03 sec)

root@127.0.0.1 : gangshen 07:37:36> INSERT INTO query_rewrite.rewrite_rules (pattern, replacement) VALUES('SELECT * from t', 'insert into t values(3,"aa")');
Query OK, 1 row affected (0.01 sec)

root@127.0.0.1 : gangshen 07:39:12> CALL query_rewrite.flush_rewrite_rules();
ERROR 1644 (45000): Loading of some rule(s) failed.
root@127.0.0.1 : gangshen 07:39:26> select * from query_rewrite.rewrite_rules;
+----+-----------------+------------------+------------------------------+---------+--------------------------------------------------+------------------------------------------------------------------+--------------------+
| id | pattern         | pattern_database | replacement                  | enabled | message                                          | pattern_digest                                                   | normalized_pattern |
+----+-----------------+------------------+------------------------------+---------+--------------------------------------------------+------------------------------------------------------------------+--------------------+
|  2 | SELECT ?        | NULL             | SELECT ? + 1                 | YES     | NULL                                             | d1b44b0c19af710b5a679907e284acd2ddc285201794bc69a2389d77baedddae | select ?           |
|  3 | SELECT * from t | NULL             | insert into t values(3,"aa") | YES     | Parse error in pattern: >>No database selected<< | NULL                                                             | NULL               |
+----+-----------------+------------------+------------------------------+---------+--------------------------------------------------+------------------------------------------------------------------+--------------------+
2 rows in set (0.00 sec)

root@127.0.0.1 : gangshen 07:39:36> update query_rewrite.rewrite_rules set pattern='SELECT * from gangshen.t' where id = 3;
Query OK, 1 row affected (0.01 sec)
Rows matched: 1  Changed: 1  Warnings: 0

root@127.0.0.1 : gangshen 07:40:18> update query_rewrite.rewrite_rules set replacement='insert into gangshen.t values(3,"aa")' where id = 3;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

root@127.0.0.1 : gangshen 07:40:50> CALL query_rewrite.flush_rewrite_rules();
Query OK, 1 row affected (0.01 sec)

root@127.0.0.1 : gangshen 07:40:54> select * from gangshen.t limit 5;
Empty set (0.00 sec)

root@127.0.0.1 : gangshen 07:41:55> select * from gangshen.t;
Query OK, 1 row affected, 1 warning (0.00 sec)

Note (Code 1105): Query 'select * from gangshen.t' rewritten to 'insert into gangshen.t values(3,"aa")' by a query rewrite plugin
root@127.0.0.1 : gangshen 07:42:00> select * from gangshen.t limit 5;
+----+------+
| id | name |
+----+------+
|  3 | aa   |
+----+------+
1 row in set (0.00 sec)
```

## Query Rewrite Plugin 的限制

- 8.0.12 之前的版本只支持 SELECT 语句的改写，8.0.12 版本开始支持 SELECT/INSERT/REPLACE/UPDATE/DELETE 语句的改写
- 只支持单独语句以及 prepare 语句的改写，视图或者存储过程相关的语句无法改写
- 改写没有语句类型的限制，比如 SELECT 语句可以改写为 INSERT 语句

## Query Rewrite Plugin 场景及总结

个人能想到的 SQL 改写可以使用到的场景：

1. 对于危险 SQL 比如 delete from table_name 这种，可以做到拦截，避免不小心将表数据完全删除
2. 业务上线后，发现某条 SQL 对数据库造成很大压力影响，可以将该 SQL 改写为 SELECT 1 这种语句，临时降低对数据库的负载
3. 数据库升级之后，SQL 执行计划不正确导致数据库负载过高，在不修改业务代码的情况下，可以通过 SQL 改写的方式让 SQL 走更优的执行计划
4. 历史库场景，过滤 DELETE 操作
5. SQL 改写可以理解为简单的触发器功能，可以参考 pt-osc 的方式支持在线字段变更
