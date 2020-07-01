---
title: TiDB + HAProxy 配置透传 IP
date: 2020-07-01 16:43:13
tags:
- TiDB
- HAProxy
categories:
- TiDB
---
## 环境介绍

* TiDB 版本：v4.0.0
* HAProxy 版本：1.5.18
* IP 信息：
 * tidb-server IP: 172.16.5.189:14000
 * HAProxy IP: 172.16.5.171:12345
 * mysql client IP：172.16.5.169

## 配置步骤

配置 HAProxy 透传 IP ，主要是需要在 haproxy 配置文件中配置 `send-proxy` 选项，以及设置 tidb 配置 `proxy-protocol.networks` 为 HAProxy 所在机器IP

* 查看集群信息

```
[tidb@node5169 gangshen]$ tiup cluster display sg-latest
Starting component `cluster`: /home/tidb/.tiup/components/cluster/v1.0.7/tiup-cluster display sg-latest
TiDB Cluster: sg-latest
TiDB Version: v4.0.0
ID                  Role        Host          Ports        OS/Arch       Status  Data Dir                                           Deploy Dir
--                  ----        ----          -----        -------       ------  --------                                           ----------
172.16.5.189:13000  grafana     172.16.5.189  13000        linux/x86_64  Up      -                                                  /home/tidb/gangshen/install/deploy/grafana-13000
172.16.4.235:12379  pd          172.16.4.235  12379/12380  linux/x86_64  Up|L    /home/tidb/gangshen/install/data/pd-12379          /home/tidb/gangshen/install/deploy/pd-12379
172.16.4.237:12379  pd          172.16.4.237  12379/12380  linux/x86_64  Up|UI   /home/tidb/gangshen/install/data/pd-12379          /home/tidb/gangshen/install/deploy/pd-12379
172.16.5.189:12379  pd          172.16.5.189  12379/12380  linux/x86_64  Up      /home/tidb/gangshen/install/data/pd-12379          /home/tidb/gangshen/install/deploy/pd-12379
172.16.5.189:19090  prometheus  172.16.5.189  19090        linux/x86_64  Up      /home/tidb/gangshen/install/data/prometheus-19090  /home/tidb/gangshen/install/deploy/prometheus-19090
172.16.5.189:14000  tidb        172.16.5.189  14000/20080  linux/x86_64  Up      -                                                  /home/tidb/gangshen/install/deploy/tidb-14000
172.16.5.171:30160  tikv        172.16.5.171  30160/30180  linux/x86_64  Up      /home/tidb/gangshen/install/data/tikv-30160        /home/tidb/gangshen/install/deploy/tikv-30160
172.16.5.172:30160  tikv        172.16.5.172  30160/30180  linux/x86_64  Up      /home/tidb/gangshen/install/data/tikv-30160        /home/tidb/gangshen/install/deploy/tikv-30160
```

* 修改 tidb 配置 `proxy-protocol.networks` 为 HAProxy 所在机器IP并 reload 重启生效

```
tiup cluster edit-config sg-latest
tiup cluster reload sg-latest -R tidb
```
![](https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20200701213442.png)


* 修改 haproxy 配置，在 backend server 配置中添加 `send-proxy` 选项

具体 haproxy 安装以及配置可以参考 TiDB 官网 [HAProxy 在 TiDB 中的最佳实践](https://docs.pingcap.com/zh/tidb/v4.0/haproxy-best-practices#%E5%90%AF%E5%8A%A8-haproxy)

![](https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20200701213525.png)

* 修改 haproxy 配置之后，重启 haproxy 生效配置

## 验证

在 172.16.5.169 机器上用 mysql client 连接 haproxy 并通过 `show processlist` 查看连接来源 IP

![](https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20200701213551.png)

## 常见问题
### 连接报 `ERROR 2013 (HY000): Lost connection to MySQL server at 'reading initial communication packet', system error: 0`
问题现象：
![](https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20200701213624.png)
问题原因：
haproxy 配置中没有配置 `send-proxy` 选项，修改 haproxy 配置之后正常。


