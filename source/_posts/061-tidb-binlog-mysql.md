---
title: 使用 TiDB Binlog 同步数据到 MySQL 步骤
date: 2022-02-14 19:39:46
tags:
- TiDB
- TiDB Binlog
categories:
- TiDB
---

# 使用 TiDB Binlog 同步数据到 MySQL 步骤

本文档适用于 TiDB 集群已经搭建完成并且已有部分数据在集群中，需要搭建 TiDB Binlog 将 TiDB 中数据同步到 MySQL 的情况 。

**说明：**
* $ 符号表示命令需要在服务器命令行中执行
* ${xxx} 表示需要替换成实际对应的内容

## 步骤一、扩容 pump 节点

1. 准备扩容 pump 节点的拓扑文件

```shell
$ cat scale-pump.yaml
pump_servers:
  host: 10.55.12.31
  host: 10.55.12.32
  host: 10.55.12.33
```

2. 执行扩容命令

```shell
$ tiup cluster scale-out ${cluster_name} ./scale-pump.yaml -uroot -p
```

3. 修改 TiDB 集群，开启 Binlog 

```shell
$ tiup cluster edit-config ${cluster_name}
在 server_config 部分添加对应的配置
server_configs:
  tidb:
    binlog.enable: true
    binlog.ignore-error: true
```

4. 滚动重启 tidb-server 节点开始 binlog

```shell
$ tiup cluster reload ${cluster_name} -R tidb
```

5. 查看集群状态

```shell
$ tiup cluster display ${cluster_name}
```

## 步骤二、全量数据迁移

1. 使用 mydumper 全量初始化导出

```shell
$ wget http://download.pingcap.org/tidb-enterprise-tools-latest-linux-amd64.tar.gz
$ tar -xzf tidb-enterprise-tools-latest-linux-amd64.tar.gz && cd tidb-enterprise-tools-latest-linux-amd64
$ ./bin/mydumper -h ${source ip} -P 4000 -u root -t 16 -F 64 --skip-tz-utc -o ./tmp/all_backup
```

2. 获取 initial_commit_ts

```shell
$ cat /tmp/all_backup/metadata
Started dump at: 2019-12-13 15:16:44
SHOW MASTER STATUS:
           Log: tidb-binlog
           Pos: 413196983854432259
           GTID:
Finished dump at: 2019-12-13 15:16:48
```

Pos 413196983854432259 即为备份的 tso 时刻点值，后续 Binlog 增量同步时需要从这个时间点开始同步

3. 备份数据复制到 MySQL 节点并使用 loader 将全备数据导入到目标库中

```shell
$ ./bin/loader -h ${target ip} -u root -P 3306 -t 32 -d /tmp/all_backup 
```

## 步骤三、扩容 drainer 节点

```shell
1. 准备扩容 drainer 节点的拓扑文件
$ cat scale-drainer.yaml
drainer_servers:
  - host: 10.0.1.12
    # commit_ts 填写 mydumper 备份文件中 metadata 记录的 tso 号信息
    commit_ts: 413196983854432259
    config:
      syncer.db-type: "mysql"
      syncer.to.host: "10.0.1.12"
      syncer.to.user: "root"
      syncer.to.password: ""
      syncer.to.port: 3306
      syncer.relay.log-dir: "/dir/to/save/log"
      syncer.relay.max-file-size: 10485760
```


2. 通过 tiup 执行扩容命令

```shell
$ tiup cluster scale-out ${cluster_name} ./scale-drainer.yaml -uroot -p
```

3. 查看集群节点状态

```shell
$ tiup cluster display ${cluster_name}
```

## 步骤四、检查增量数据同步情况
在上游 TiDB 执行一些变更操作，在下游 MySQL 中查询验证数据是否正常同步。
