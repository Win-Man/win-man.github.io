---
title: MySQL增强（Loss-less）半同步复制
date: 2018-08-10 17:12:13
tags:
- MySQL
- Replication
categories:
- MySQL
---

> 原文地址：http://my-replication-life.blogspot.com/2013/09/loss-less-semi-synchronous-replication.html?view=classic
> 原文作者： Sunny Bains, Jiamin Huang (University of Michigan)
> 译者：沈刚


## 前言
许多用户在他们的数据库环境中使用半同步复制架构以提高数据的完整性。今天我要向大家介绍一下MySQL在5.7版本中针对半同步复制增加的新特性，这个新特性进一步增强了主库和备库之间的数据完整性。

## 在存储引擎层提交之前等待从库ACK
正如你所知的，在半同步复制架构中，主库上的事务提交之前会等待一个从库返回ACK信号。为了进一步增加数据的完整性，新的特性将等待从库返回ACK信号的时间点提前了（相对于MySQL5.5以及MySQL5.6），新特性中主库上的事务会在存储引擎层提交之前一直等待从库返回ACK信号。
![](http://2.bp.blogspot.com/-6aMYLMuhBmg/UjKOkzQcrMI/AAAAAAAAABw/Z9S9GKszGGs/s640/after_sync.jpg)

有了这个新特性之后，半同步复制可以保证：

* 在主库crash的情况下，所有在主库上已经提交的事务已经被复制到至少一个从库上

显而易见，因为主库上的事务无法提交，除非从库返回了ACK信号（或者超时）。
这对于用户来说有两个有利点：

* 强数据完整性，并解决了幻读的问题
* 简化主库Crash Recovery的过程

## 强数据完整性解决幻读问题
在MySQL 5.5以及MySQL 5.6开启半同步复制的场景下，主库上的事务在存储引擎层提交之后，需要等待从库返回ACK信号。并且在接收到从库返回ACK信号或者等待超时才会返回给客户端一个提交结果。
![](http://3.bp.blogspot.com/-1mGL5sMoRIs/UjKCpOBUVQI/AAAAAAAAABg/wV7_pHwCGis/s640/after_commit.jpg)

存储引擎层的提交，会持久化记录并且释放这些记录上的锁。所以在存储引擎层提交之后，其余的会话可以操作并读取这些记录，即使这个会话还在等待从库返回ACK信号。这样在主库crash，从库接管主库的情况下，产生幻读的现象。
![](http://4.bp.blogspot.com/-7wnqu8CB7B4/UjKpRoc7COI/AAAAAAAAACA/X9V0SPOf54o/s400/failover.jpg)

在这个新特性中，不会发生幻读。User2在第一次SELECT的时候不会获取3这个值，因为3这条记录还没有复制从库，所以并没有在存储引擎层提交。

## 简化主库Crash Recovery的过程
在MySQL 5.7.2之前的版本中，用户在恢复crash掉的主库的时候，需要做以下操作：

* 手动清除并没有被复制到从库上的binlog事务
* 手动回滚已经提交但是还没有被复制的事务

因为新的特性保证所有的事务在提交之前都至少复制到一个从库上了，所以第二步可以不用做了。

## 如何设置新特性
这个特性的设置很简单，用户无需任何设置，因为在MySQL 5.7.2版本之后该特性是默认开启的。用户可以设置`rpl_semi_sync_master_wait_point`变量控制主库等待从库返回ACK信号的时间点。

这是一个全局、可动态修改的参数。在5.7.2版本之后，该变量的默认值是`AFTER_SYNC`，该值表示，主库上的事务会在flush binlog之后，在存储引擎层提交之前的时间点，等待从库返回ACK信号。

你也可以将该参数设置为`AFTER_COMMIT`值不开启新特性，保持和之前版本一样的机制。但是我想不到一个理由让用户设置该变量为`AFTER_COMMIT`值。开启新特性并没有副作用，也不会对数据库性能产生影响。

## Dump线程的优化
在开发这个特性的过程中，我们对Dump线程做了一些优化。我们重构了Dump线程先关的代码，但是更重要的是，在Dump线程读取binlog event的时候，Dump线程不会再去获取binlog锁。这一改进增加了主库的一个吞吐量并且减小了主从之间的延迟。

## 最后
总而言之，这个新特性保证了主库和从库之间的数据完整性、一致性，并不会带来任何副作用以及性能印象。我强烈推荐你去尝试该特性。

```
A special thanks to Mr. Zhenxing Zhou a community user who contributed a patch implementing a similar idea based on MySQL 5.5. Even though we did not take his patch (the 5.7 codebase changed quite a lot compared to 5.5), his feature request was yet another source of motivation and inspiration.
```

