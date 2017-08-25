---
title:NFS总结
date: 2017-08-25 16:13:04
tags:
- Linux
categories:
- Linux
---
## 什么是NFS，为什么要NFS
NFS（Network File System）这个网络文件系统，最大的功能是通过网络，使得在不同的机器之间共享文件。通过NFS可以将另一台机器的某个目录挂载到本机的某个目录下，在使用的时候挂载过来的目录就像在本地一样。极大的方便了多台服务器之间的文件共享。

## 什么是RPC
在提到NFS服务的时候，都会提到RPC服务，那么什么是RPC服务，为什么需要RPC服务呢。
首先RPC是Remote Procedure Call的缩写，及远程过程调用。其次关于为什么要RPC服务是因为，NFS服务在启动的时候对于端口的选择是随机的，NFS在启动的时候回随机选择小于1024的端口进行数据的传输。那么在客户端连接到NFS服务器的时候，怎么知道应该通过哪个端口进行连接呢。这个时候就需要RPC服务出现了，RPC的主要功能就是记录每个NFS服务所对应的端口号，并将对应的端口号发送给客户端，客户端根据获得的端口号进行连接。
**客户端连接NFS服务的过程：**
1. 客户端首先向NFS服务所在的服务器的RPC服务发起请求，RPC服务的端口固定为111
2. RPC服务在接收到客户端的请求之后，找到注册的NFS服务所对应的端口号，并将端口号发送给客户端
3. 客户端根据获取到NFS端口号进行连接

## 配置NFS步骤
* 环境介绍
 * CentOS6.6

### NFS主要软件结构
* /etc/exports NFS的主要配置文件，在该文件中定义NFS输出的目录、权限等属性
* /usr/sbin/exportfs 管理NFS共享资源的命令，在NFS的server端使用
* /usr/sbin/showmount 查询NFS共享资源的命令，在client端使用
* /var/lib/nfs/etab NFS的日志文件，记录了NFS共享出来的目录的完整权限设置值
* /var/lib/nfs/xtab NFS连接信息的日志文件，记录了曾经链接到此NFS服务器的相关客户端的数据

### /etc/exports配置文件介绍
配置文件中的格式：
```
<输出目录> [客户端1(权限)] [客户端2(权限)] ...
```
* 输出目录
指定共享的文件目录

* 客户端
指定可以访问共享资源的客户端
 * 指定IP的方式：192.168.1.1
 * 指定网段的方式 192.168.1.0/24或者192.168.1.0/255.255.255.0
 * 指定主机名的方式 node1，但是这个主机名必须在记录在/etc/hosts文件内，需要能解析到对应的ip
 * 通配符的方式 *表示所有的主机，node*主机名通配

* 权限
 * rw/ro 指定文件是可读写（read-write）的还是只读（read-only）的
 * sync/async 对目录中的数据进行更改的话，更改会同步写到硬盘还是只写到内存直接写到硬盘
 * no_root_squash 当客户端使用root账户连接的时候，不转化为匿名用户
 * root_squash 当客户端使用root账户连接的会后，转化为匿名用户
 * all_squash 当客户端连接的时候，无论使用什么用户进行连接，都会被转换为匿名用户
 * anonuid 指定匿名用户的UID
 * anongid 指定匿名用户的GID
 
### NFS server端配置

* 检查服务器是否安装了nfs-util和rpcbind两个包
```
[root@d146bbc8-94a2-42c4-8eaa-3fb3264fd176 ~]# rpm -qa | grep nfs
nfs-utils-1.2.3-54.el6.x86_64
nfs-utils-lib-1.1.5-9.el6.x86_64
[root@d146bbc8-94a2-42c4-8eaa-3fb3264fd176 ~]# rpm -qa | grep rpcbind
rpcbind-0.2.0-11.el6.x86_64
```

* 填写/etc/exports配置文件
```
mkdir -p /nfs/{test1,test2,test3,test4,test5}
vim /etc/exports
```
![](http://oc4wmeyj8.bkt.clouddn.com/nfs.png)
exports配置文件中都有默认权限配置，如果我们想查看默认的配置是什么，可以查看`/var/lib/nfs/etab`文件，该文件中完整的记录了NFS共享资源的所有配置
```
/nfs/test2	10.10.30.247(rw,sync,wdelay,hide,nocrossmnt,secure,root_squash,no_all_squash,no_subtree_check,secure_locks,acl,anonuid=65534,anongid=65534)
/nfs/test1	10.10.30.247(rw,sync,wdelay,hide,nocrossmnt,secure,root_squash,no_all_squash,no_subtree_check,secure_locks,acl,anonuid=65534,anongid=65534)
/nfs/test3	10.10.30.0/24(rw,sync,wdelay,hide,nocrossmnt,secure,root_squash,no_all_squash,no_subtree_check,secure_locks,acl,anonuid=65534,anongid=65534)
/nfs/test2	*(ro,sync,wdelay,hide,nocrossmnt,secure,root_squash,no_all_squash,no_subtree_check,secure_locks,acl,anonuid=65534,anongid=65534)
```

* 启动RPC服务
```
[root@d146bbc8-94a2-42c4-8eaa-3fb3264fd176 ~]# /etc/init.d/rpcbind start
```

* 启动NFS服务
```
[root@d146bbc8-94a2-42c4-8eaa-3fb3264fd176 ~]# /etc/init.d/nfs start
Starting NFS services:                                     [  OK  ]
Starting NFS quotas:                                       [  OK  ]
Starting NFS mountd:                                       [  OK  ]
Starting NFS daemon:                                       [  OK  ]
Starting RPC idmapd:                                       [  OK  ]
```

* 修改/etc/exports文件
如果我们在启动NFS服务之后，又修改了 `exports` 文件的话怎么办，是不是需要重新启动NFS服务才能使的这些配置生效，答案当然是否定的，这样多麻烦。我们有 `exportfs` 命令。
```
exportfs [-aruv]

options:
 -a:全部挂载或卸载
 -r:重新挂载/etc/exports配置文件中的信息
 -u:卸载某一目录
 -v:将内容显示输出
```

### client端配置

* 通过showmount命令查看NFS共享资源的情况
```
[root@6a192e7f-a43d-4b57-9573-ca740e3b5f0d ~]# showmount -e 10.10.30.250
Export list for 10.10.30.250:
/nfs/test3 10.10.30.0/24
/nfs/test2 (everyone)
/nfs/test1 10.10.30.247
```
```
showmount [-ae] [hostname|IP]

options:
 -a:显示当前主机与客户端的NFS连接情况
 -e:显示指定NFS服务器的共享资源情况
```

* 挂载NFS目录
```
[root@6a192e7f-a43d-4b57-9573-ca740e3b5f0d ~]# mkdir -p /nfs/local/{test1,test2,test3}
[root@6a192e7f-a43d-4b57-9573-ca740e3b5f0d ~]# mount 10.10.30.250:/nfs/test1 /nfs/local/test1
[root@6a192e7f-a43d-4b57-9573-ca740e3b5f0d ~]# mount 10.10.30.250:/nfs/test2 /nfs/local/test2
[root@6a192e7f-a43d-4b57-9573-ca740e3b5f0d ~]# mount 10.10.30.250:/nfs/test3 /nfs/local/test3
```

* 查看挂载情况
```
[root@6a192e7f-a43d-4b57-9573-ca740e3b5f0d ~]# df -h
Filesystem            Size  Used Avail Use% Mounted on
/dev/mapper/VolGroup-root
                       50G  5.3G   42G  12% /
tmpfs                 1.9G     0  1.9G   0% /dev/shm
/dev/vda1             488M   31M  432M   7% /boot
/dev/sda               20G   33M   20G   1% /data
10.10.30.250:/nfs/test1
                       50G  5.3G   42G  12% /nfs/local/test1
10.10.30.250:/nfs/test2
                       50G  5.3G   42G  12% /nfs/local/test2
10.10.30.250:/nfs/test3
                       50G  5.3G   42G  12% /nfs/local/test3
[root@6a192e7f-a43d-4b57-9573-ca740e3b5f0d ~]# mount
/dev/mapper/VolGroup-root on / type ext4 (rw)
proc on /proc type proc (rw)
sysfs on /sys type sysfs (rw)
devpts on /dev/pts type devpts (rw,gid=5,mode=620)
tmpfs on /dev/shm type tmpfs (rw)
/dev/vda1 on /boot type ext4 (rw)
none on /proc/sys/fs/binfmt_misc type binfmt_misc (rw)
/dev/sda on /data type xfs (rw,nouuid)
sunrpc on /var/lib/nfs/rpc_pipefs type rpc_pipefs (rw)
10.10.30.250:/nfs/test1 on /nfs/local/test1 type nfs (rw,vers=4,addr=10.10.30.250,clientaddr=10.10.30.247)
10.10.30.250:/nfs/test2 on /nfs/local/test2 type nfs (rw,vers=4,addr=10.10.30.250,clientaddr=10.10.30.247)
10.10.30.250:/nfs/test3 on /nfs/local/test3 type nfs (rw,vers=4,addr=10.10.30.250,clientaddr=10.10.30.247)
```

## NFS文件访问权限设置
我们知道在Linux系统中的文件，都有访问权限，可以指定用户访问，那么我通过NFS访问文件权限是如何设定的呢。比如我再NFS server上有一个文件的属主与属组都是gangshen,那我再client端普通用户user1访问这个文件，可以做修改吗？如果是使用root用户访问这个文件，可以做修改吗？
NFS文件系统的权限访问的原则是：**服务器端会以客户端的用户UID与GID等身份来尝试读取服务器端的文件系统**
以下是NFS是如何判断是否一个文件可写的步骤:
1. 客户端以user1这个账户访问NFS目录，实际是以client端的user1的UID以及GID进行访问的
2. NFS server收到客户端访问请求中的UID、GID，并根据这个UID在server端所在的服务器找到相应的用户，以该用户的权限进行访问
3. 如果找不到对应UID的用户，则NFS server端会将连接的身份转换为匿名用户的连接
4. 根据/etc/exports配置文件判断该目录是否有写入权限
只有在满足：1、用户有写入权限 2、exports设置为有写入权限 3、文件具有写入权限 这三点之后，客户端才可以写入NFS是共享目录。


根据上述步骤的话，可以想到一种情况，如果我client端是通过root账户连接的，那我是不是对所有的文件具有可读、可写权限。这个的话，就需要重新来看下/etc/exports这个配置文件了。在exports文件中，设定权限的时候可以指定no_root_squash、root_squash、all_squash，这三个就是控制，我client连接上来的用户，最终在server端会以哪个用户进行访问。默认情况下是设置为root_squash，就是说如果client以root用户连接上来的话，server端会将该用户转换为匿名用户进行访问。很明显no_root_squash这个选项就是说不转换成匿名用户，all_squash这个选项是指定无论client用什么账户连接上来，都会被转换成匿名用户。








