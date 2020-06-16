---
title: TiDB 异常断电恢复后 PD 节点启动失败
date: 2020-06-16 16:43:13
tags:
- TiDB
- 故障恢复
categories:
- TiDB
---

## 环境信息及故障现象

### 集群版本

v4.0.0

### 故障现象

TiDB 集群的物理机异常断电重启,机器恢复后,通过 `tiup cluster start <cluster-name>` 启动集群，发现有两个 `PD` 节点启动失败

```shell
[root@localhost ~]# tiup cluster start t11
Starting component `cluster`: /root/.tiup/components/cluster/v1.0.0/cluster start t11
Starting cluster t11...
+ [ Serial ] - SSHKeySet: privateKey=/root/.tiup/storage/cluster/clusters/t11/ssh/id_rsa, publicKey=/root/.tiup/storage/cluster/clusters/t11/ssh/id_rsa.pub
+ [Parallel] - UserSSH: user=tidb, host=192.168.73.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.73.151
+ [Parallel] - UserSSH: user=tidb, host=192.168.73.152
+ [Parallel] - UserSSH: user=tidb, host=192.168.73.153
+ [Parallel] - UserSSH: user=tidb, host=192.168.73.151
+ [Parallel] - UserSSH: user=tidb, host=192.168.73.152
+ [Parallel] - UserSSH: user=tidb, host=192.168.73.155
+ [Parallel] - UserSSH: user=tidb, host=192.168.73.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.73.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.73.154
+ [Parallel] - UserSSH: user=tidb, host=192.168.73.153
+ [Parallel] - UserSSH: user=tidb, host=192.168.73.128
+ [ Serial ] - ClusterOperate: operation=StartOperation, options={Roles:[] Nodes:[] Force:false SSHTimeout:5 OptTimeout:60 APITimeout:300}
Starting component pd
	Starting instance pd 192.168.73.153:2379
	Starting instance pd 192.168.73.151:2379
	Starting instance pd 192.168.73.152:2379
	Start pd 192.168.73.151:2379 success
retry error: operation timed out after 1m0s
	pd 192.168.73.153:2379 failed to start: timed out waiting for port 2379 to be started after 1m0s, please check the log of the instance
retry error: operation timed out after 1m0s
	pd 192.168.73.152:2379 failed to start: timed out waiting for port 2379 to be started after 1m0s, please check the log of the instance

Error: failed to start: failed to start pd: 	pd 192.168.73.153:2379 failed to start: timed out waiting for port 2379 to be started after 1m0s, please check the log of the instance: timed out waiting for port 2379 to be started after 1m0s

Verbose debug logs has been written to /root/logs/tiup-cluster-debug-2020-06-16-16-28-39.log.
Error: run `/root/.tiup/components/cluster/v1.0.0/cluster` (wd:/root/.tiup/data/S23ruRB) failed: exit status 1
```

登陆到 `PD` 节点所在机器，查看 `pd-server` 进程的确不存在，通过 `pd.log` 日志查看 `PD` 启动失败的原因：

```
[2020/06/16 16:29:10.180 +08:00] [INFO] [systime_mon.go:26] ["start system time monitor"]
[2020/06/16 16:29:10.181 +08:00] [INFO] [backend.go:79] ["opened backend db"] [path=/tidb/tidb-data/pd-2379/member/snap/db] [took=739.112µs]
[2020/06/16 16:29:10.182 +08:00] [INFO] [server.go:443] ["recovered v2 store from snapshot"] [snapshot-index=1400015] [snapshot-size="22 kB"]
[2020/06/16 16:29:10.195 +08:00] [WARN] [db.go:92] ["failed to find [SNAPSHOT-INDEX].snap.db"] [snapshot-index=1400015] [snapshot-file-path=/tidb/tidb-data/pd-2379/member/snap/0000000000155ccf.snap.db] [error="snap: snapshot file doesn't exist"]
[2020/06/16 16:29:10.195 +08:00] [PANIC] [server.go:454] ["failed to recover v3 backend from snapshot"] [error="failed to find database snapshot file (snap: snapshot file doesn't exist)"]
[2020/06/16 16:29:10.195 +08:00] [FATAL] [log.go:292] [panic] [recover="\"invalid memory address or nil pointer dereference\""] [stack="github.com/pingcap/log.Fatal\n\t/home/jenkins/agent/workspace/build_pd_multi_branch_v4.0.0/go/pkg/mod/github.com/pingcap/log@v0.0.0-20200117041106-d28c14d3b1cd/global.go:59\ngithub.com/pingcap/pd/v4/pkg/logutil.LogPanic\n\t/home/jenkins/agent/workspace/build_pd_multi_branch_v4.0.0/go/src/github.com/pingcap/pd/pkg/logutil/log.go:292\nruntime.gopanic\n\t/usr/local/go/src/runtime/panic.go:679\nruntime.panicmem\n\t/usr/local/go/src/runtime/panic.go:199\nruntime.sigpanic\n\t/usr/local/go/src/runtime/signal_unix.go:394\ngo.etcd.io/etcd/etcdserver.NewServer.func1\n\t/home/jenkins/agent/workspace/build_pd_multi_branch_v4.0.0/go/pkg/mod/go.etcd.io/etcd@v0.5.0-alpha.5.0.20191023171146-3cf2f69b5738/etcdserver/server.go:335\nruntime.gopanic\n\t/usr/local/go/src/runtime/panic.go:679\ngo.uber.org/zap/zapcore.(*CheckedEntry).Write\n\t/home/jenkins/agent/workspace/build_pd_multi_branch_v4.0.0/go/pkg/mod/go.uber.org/zap@v1.13.0/zapcore/entry.go:230\ngo.uber.org/zap.(*Logger).Panic\n\t/home/jenkins/agent/workspace/build_pd_multi_branch_v4.0.0/go/pkg/mod/go.uber.org/zap@v1.13.0/logger.go:225\ngo.etcd.io/etcd/etcdserver.NewServer\n\t/home/jenkins/agent/workspace/build_pd_multi_branch_v4.0.0/go/pkg/mod/go.etcd.io/etcd@v0.5.0-alpha.5.0.20191023171146-3cf2f69b5738/etcdserver/server.go:454\ngo.etcd.io/etcd/embed.StartEtcd\n\t/home/jenkins/agent/workspace/build_pd_multi_branch_v4.0.0/go/pkg/mod/go.etcd.io/etcd@v0.5.0-alpha.5.0.20191023171146-3cf2f69b5738/embed/etcd.go:211\ngithub.com/pingcap/pd/v4/server.(*Server).startEtcd\n\t/home/jenkins/agent/workspace/build_pd_multi_branch_v4.0.0/go/src/github.com/pingcap/pd/server/server.go:257\ngithub.com/pingcap/pd/v4/server.(*Server).Run\n\t/home/jenkins/agent/workspace/build_pd_multi_branch_v4.0.0/go/src/github.com/pingcap/pd/server/server.go:441\nmain.main\n\t/home/jenkins/agent/workspace/build_pd_multi_branch_v4.0.0/go/src/github.com/pingcap/pd/cmd/pd-server/main.go:118\nruntime.main\n\t/usr/local/go/src/runtime/proc.go:203"]
```

看到两个 `PD` 节点都报 `failed to recover v3 backedn from snapshot` 错误。

### 故障原因分析

### 故障解决步骤

参考官方文档 [PD Recover 使用文档](https://docs.pingcap.com/zh/tidb/v4.0/pd-recover#pd-recover-%E4%BD%BF%E7%94%A8%E6%96%87%E6%A1%A3) ,通过 `pd-recovery` 工具恢复集群。

根据文档内容主要分为 3 个步骤：1. 找到 `cluster id` 2. 找到当前最大 `alloc id` 3. 通过 pd-recovery 恢复 pd 集群

这边的话，都是通过 PD 日志来查找 `cluster id` 和 `alloc id`。

**操作步骤**

1. 查找 cluster id

```
[root@localhost ~]# cat /tidb/tidb-bin/pd-2379/log/pd.log | grep "init cluster id"
[2020/05/06 23:37:02.121 +08:00] [INFO] [server.go:340] ["init cluster id"] [cluster-id=6823755660393880966]
[2020/05/07 00:03:25.132 +08:00] [INFO] [server.go:340] ["init cluster id"] [cluster-id=6823755660393880966]
[2020/05/07 11:45:39.338 +08:00] [INFO] [server.go:340] ["init cluster id"] [cluster-id=6823755660393880966]
[2020/05/25 14:54:50.076 +08:00] [INFO] [server.go:336] ["init cluster id"] [cluster-id=6823755660393880966]
[2020/05/25 16:45:55.526 +08:00] [INFO] [server.go:336] ["init cluster id"] [cluster-id=6823755660393880966]
[2020/05/25 16:48:21.462 +08:00] [INFO] [server.go:336] ["init cluster id"] [cluster-id=6823755660393880966]
[2020/06/01 19:13:17.478 +08:00] [INFO] [server.go:336] ["init cluster id"] [cluster-id=6823755660393880966]
[2020/06/04 02:28:29.655 +08:00] [INFO] [server.go:336] ["init cluster id"] [cluster-id=6823755660393880966]
[2020/06/05 22:27:46.152 +08:00] [INFO] [server.go:336] ["init cluster id"] [cluster-id=6823755660393880966]
[2020/06/08 22:50:30.045 +08:00] [INFO] [server.go:336] ["init cluster id"] [cluster-id=6823755660393880966]
[2020/06/08 22:50:59.534 +08:00] [INFO] [server.go:336] ["init cluster id"] [cluster-id=6823755660393880966]
[2020/06/11 01:48:35.936 +08:00] [INFO] [server.go:336] ["init cluster id"] [cluster-id=6823755660393880966]
```

2. 查找当前最大的 `alloc id` ，因为 pd-recovery 去恢复的时候需要指定一个比当前最大的 `alloc id` 更大的 `alloc id`，所以需要对所有的 pd 节点日志进行查找

```
pd-1
[root@localhost ~]# cat /tidb/tidb-bin/pd-2379/log/pd.log | grep "allocates"
[2020/05/06 23:37:04.752 +08:00] [INFO] [id.go:110] ["idAllocator allocates a new id"] [alloc-id=1000]
[2020/05/12 11:21:28.271 +08:00] [INFO] [id.go:110] ["idAllocator allocates a new id"] [alloc-id=2000]

pd-2
[root@localhost ~]# cat /tidb/tidb-bin/pd-2379/log/pd.log | grep "allocates"

pd-3
[root@localhost ~]# cat /tidb/tidb-bin/pd-2379/log/pd.log | grep "allocates"
[2020/05/27 11:20:20.687 +08:00] [INFO] [id.go:110] ["idAllocator allocates a new id"] [alloc-id=3000]
[2020/06/10 18:04:43.361 +08:00] [INFO] [id.go:110] ["idAllocator allocates a new id"] [alloc-id=4000]
```

3. 删除旧 PD 集群 data 目录下的所有内容，因为这个集群 PD 节点 data 目录为 /tidb/tidb-data/pd-2379 ,所以删除 /tidb/tidb-data/pd-2379 目录下所有内容

```
查看 data 目录
[root@localhost ~]# tiup cluster display t11
......
192.168.73.151:2379   pd            192.168.73.151  2379/2380                        linux/x86_64  Healthy    /tidb/tidb-data/pd-2379            /tidb/tidb-bin/pd-2379
192.168.73.152:2379   pd            192.168.73.152  2379/2380                        linux/x86_64  Healthy    /tidb/tidb-data/pd-2379            /tidb/tidb-bin/pd-2379
192.168.73.153:2379   pd            192.168.73.153  2379/2380                        linux/x86_64  Healthy|L  /tidb/tidb-data/pd-2379            /tidb/tidb-bin/pd-2379
......

删除数据目录

[root@localhost ~]# mv /tidb/tidb-data/pd-2379/* /tmp
```


4. 启动 PD 集群

```
tiup cluster start t11 -R pd
```

5. 通过 pd-recovery 恢复集群，`--endpoints` 指定一个 pd 节点，`-cluster-id` 指定查找到的 cluster-id ,`-alloc-id` 指定比查找到的最大 alloc id 更大的一个数字，所以这边只要指定一个比 4000 更大的数字即可

```
[root@localhost ~]# ./pd-recover -endpoints http://192.168.73.151:2379 -cluster-id 6823755660393880966 -alloc-id 10000
recover success! please restart the PD cluster
```

6. 重启 PD 集群

```
tiup cluster restart t11 -R pd
```

7. 启动集群

```
tiup cluster start t11
```

8. 查看集群状态，恢复正常

```
tiup cluster display t11
```

**注意**
> 通过 tiup 部署的 TiDB 集群需要用户自己下载 pd-recovery 工具，可以通过 `http://download.pingcap.org/tidb-{{ tidb_version }}-linux-amd64.tar.gz` 链接进行下载