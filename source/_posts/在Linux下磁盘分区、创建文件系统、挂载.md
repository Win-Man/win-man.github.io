---
title: 在Linux下磁盘分区、创建文件系统、挂载
date: 2017-07-11 14:52:22
tags:
- Linux
- 磁盘
- 分区
categories:
- Linux
---
## 简介
学习Linux的时候，避不开的就是需要学习Linux中的分区。学习Linux的分区，有很多概念性的东西需要学习，比如分区类型啊，文件系统类型啊...等等，对于我来说，并不是很喜欢这种概念性的东西，上来一大段文字对于我一点都不友好。所以整理了一篇笔记，记录怎么对Linux中的磁盘进行分区、创建文件系统以及挂载。首先能实操使用，接着再慢慢理解其中的内容。
### 环境介绍
操作系统：CentOS6.6

## 命令介绍

### 分区命令
#### fdisk 分区
fdisk可用于查看硬盘分区情况，也可以使用fdisk进行分区。
**选项：**
```
-b SECTOR_SIZE：指定每个分区大小
-l:列出指定硬盘的分区情况
-s PARTITION:输出指定分区的大小，单位为块
-u:在列出硬盘分区情况的时候，使用块为单位列出分区大小而不是以柱面为单位
-v:显示版本信息
```
**子选项：**
```
p:print，显示已有分区
n:new，创建
d:delete，删除
w:write，写入磁盘并退出
q:quit，放弃更新并退出
m:获取帮助
l:列表所分区id
t:调整分区id
```
#### partx 识别分区
***选项：*
```
-a:通知内核重新读取硬盘分区表
-d:删除分区硬盘分区表信息
-l:列出分区
--type YTPE:指定分区类型
--nr M-N:指定分区范围
```

### 创建文件系统命令

#### mkfs 创建文件系统
**选项：**
```
-t TYPE:指定穿件的文件系统类型
-v:显示版本信息
-c:创建文件系统之前，先检查分区是否存在坏块
```

#### mkswap 创建交换分区
**选项：**
```
-c:创建交换分区之前，先检查分区是否存在坏块
-f:强制执行
```

### 挂载命令

#### mount 挂载分区
mount [-fnrsvw] [-t vfstype] [-o options] device dir
**选项：**
```
-t vsftype:指定要挂载的设备上的文件系统类型
-r ：readonly，只读挂载
-w：read and write，读写挂载
-n：不更新/etc/fstab
-a：自动挂载所有支持自动挂载的设备（定义在了/etc/fstab文件中，且挂载选项中有“自动挂载”功能）
-L 'LABEL'：以卷标指定挂载设备
-U 'UUID'：以UUID指定要挂载的设备
-B，--bind：绑定目录到另一个目录上


-o options:挂载文件系统的选项
async：异步模式
sync：同步模式
atime/noatime:包含目录和文件
diratime/nodiratime:目录的访问时间戳
auto/noauto:是否支持自动挂载
exec/noexec:是否支持将文件系统上的应用程序运行为进程
dev/nodev:是否支持在此文件系统上使用设备
suid/nosuid；
remount：重新挂载
ro：只读挂载
rw：读写挂载
user/nouser:是否允许普通用户挂载此设备
acl：启用此文件系统上acl功能
defaults：rw,suid,dev,exec,auto,nouser,and async
```
#### umount 卸载分区
umount [-dflnrv] {dir|device}...
**选项：**
```
-a：卸除/etc/mtab中记录的所有文件系统；
-h：显示帮助；
-n：卸除时不要将信息存入/etc/mtab文件中；
-r：若无法成功卸除，则尝试以只读的方式重新挂入文件系统；
-t<文件系统类型>：仅卸除选项中所指定的文件系统；
-v：执行时显示详细的信息；
-V：显示版本信息。
```

#### swapon 启用交换分区
swapon [-f] [-p priority] [-v] specialfile...
**选项：**
```
-a ：激活所有的交换分区
-p PRIORITY：指定优先级
```

#### swapoff 禁用交换分区
swapoff [-v] specialfile...
**选项：**
```
-a：禁用所有的交换分区
```

## 步骤介绍

### 普通分区从创建以及挂载
* 查看硬盘以及分区情况
```
[root@sg-pc /]# lsblk
NAME                       MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sr0                         11:0    1  4.3G  0 rom  
sda                          8:0    0   20G  0 disk 
├─sda1                       8:1    0  500M  0 part /boot
└─sda2                       8:2    0 19.5G  0 part 
  ├─vg_sgpc-lv_root (dm-0) 253:0    0 17.6G  0 lvm  /
  └─vg_sgpc-lv_swap (dm-1) 253:1    0    2G  0 lvm  [SWAP]
sdb                          8:16   0   20G  0 disk 
[root@sg-pc /]# fdisk /dev/sdb

WARNING: DOS-compatible mode is deprecated. It's strongly recommended to
         switch off the mode (command 'c') and change display units to
         sectors (command 'u').

Command (m for help): p

Disk /dev/sdb: 21.5 GB, 21474836480 bytes
255 heads, 63 sectors/track, 2610 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x179a39d6

   Device Boot      Start         End      Blocks   Id  System

Command (m for help): 
```

* 使用fdisk对/dev/sdb硬盘划分分区，划分3个主分区，每个主分区5G大小，并且划分一个扩展分区，4G大小。
```
[root@sg-pc /]# fdisk /dev/sdb

WARNING: DOS-compatible mode is deprecated. It's strongly recommended to
         switch off the mode (command 'c') and change display units to
         sectors (command 'u').

Command (m for help): n
Command action
   e   extended
   p   primary partition (1-4)
p
Partition number (1-4): 1
First cylinder (1-2610, default 1): 
Using default value 1
Last cylinder, +cylinders or +size{K,M,G} (1-2610, default 2610): +5G

Command (m for help): n
Command action
   e   extended
   p   primary partition (1-4)
p
Partition number (1-4): 2
First cylinder (655-2610, default 655): 
Using default value 655
Last cylinder, +cylinders or +size{K,M,G} (655-2610, default 2610): +5G

Command (m for help): n
Command action
   e   extended
   p   primary partition (1-4)
p
Partition number (1-4): 3
First cylinder (1309-2610, default 1309): 
Using default value 1309
Last cylinder, +cylinders or +size{K,M,G} (1309-2610, default 2610): +5G

Command (m for help): n
Command action
   e   extended
   p   primary partition (1-4)
e
Selected partition 4
First cylinder (1963-2610, default 1963): 
Using default value 1963
Last cylinder, +cylinders or +size{K,M,G} (1963-2610, default 2610): +2610
Value out of range.
Last cylinder, +cylinders or +size{K,M,G} (1963-2610, default 2610): 
Using default value 2610

Command (m for help): p

Disk /dev/sdb: 21.5 GB, 21474836480 bytes
255 heads, 63 sectors/track, 2610 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x179a39d6

   Device Boot      Start         End      Blocks   Id  System
/dev/sdb1               1         654     5253223+  83  Linux
/dev/sdb2             655        1308     5253255   83  Linux
/dev/sdb3            1309        1962     5253255   83  Linux
/dev/sdb4            1963        2610     5205060    5  Extended

Command (m for help): n
First cylinder (1963-2610, default 1963): 
Using default value 1963
Last cylinder, +cylinders or +size{K,M,G} (1963-2610, default 2610): +2G 

Command (m for help): p

Disk /dev/sdb: 21.5 GB, 21474836480 bytes
255 heads, 63 sectors/track, 2610 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x179a39d6

   Device Boot      Start         End      Blocks   Id  System
/dev/sdb1               1         654     5253223+  83  Linux
/dev/sdb2             655        1308     5253255   83  Linux
/dev/sdb3            1309        1962     5253255   83  Linux
/dev/sdb4            1963        2610     5205060    5  Extended
/dev/sdb5            1963        2224     2104483+  83  Linux

Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.
```
* 将新划分的分区加载到硬盘分区表中，并查看分区状态
```
[root@sg-pc /]# partx -a /dev/sdb
BLKPG: Device or resource busy
error adding partition 1
BLKPG: Device or resource busy
error adding partition 2
BLKPG: Device or resource busy
error adding partition 3
BLKPG: Device or resource busy
error adding partition 4
BLKPG: Device or resource busy
error adding partition 5
[root@sg-pc /]# cat /proc/partitions 
major minor  #blocks  name

   8        0   20971520 sda
   8        1     512000 sda1
   8        2   20458496 sda2
   8       16   20971520 sdb
   8       17    5253223 sdb1
   8       18    5253255 sdb2
   8       19    5253255 sdb3
   8       20          1 sdb4
   8       21    2104483 sdb5
 253        0   18391040 dm-0
 253        1    2064384 dm-1
```

* 为分区创建文件系统，这边就选择/dev/sdb1分区作为示例
```
[root@sg-pc /]# mkfs -t ext4 -L 'backup' /dev/sdb1
mke2fs 1.41.12 (17-May-2010)
Filesystem label=backup
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
328656 inodes, 1313305 blocks
65665 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=1346371584
41 block groups
32768 blocks per group, 32768 fragments per group
8016 inodes per group
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912, 819200, 884736

Writing inode tables: done                            
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done

This filesystem will be automatically checked every 25 mounts or
180 days, whichever comes first.  Use tune2fs -c or -i to override.
```
* 通过blkid命令，可以查看硬盘设备的信息
```
[root@sg-pc /]# blkid
/dev/sda1: UUID="5e605252-9e26-4ca0-b353-e3e6028bc6ca" TYPE="ext4" 
/dev/sda2: UUID="drR3Sx-Ht8g-nGFi-2vR3-CD3z-qSjY-PUXiye" TYPE="LVM2_member" 
/dev/mapper/vg_sgpc-lv_root: UUID="9ccc9289-ada6-48b8-82f1-42d2f7edba06" TYPE="ext4" 
/dev/mapper/vg_sgpc-lv_swap: UUID="9c4c6c97-b20d-4548-a249-8d98f6b400f1" TYPE="swap" 
/dev/sdb1: LABEL="backup" UUID="4a7b5acb-fc24-446d-b9e0-a14afbe171d9" TYPE="ext4" 
/dev/sdb2: LABEL="myswap" UUID="79f07b7d-c504-43c2-9e0f-87bee77d59b9" TYPE="swap" 
```
* 将创建完文件系统的分区挂载指定目录下
```
[root@sg-pc /]# mkdir /backup
[root@sg-pc /]# mount /dev/sdb1 /backup/
```
* 检查分区挂载情况
![](http://oc4wmeyj8.bkt.clouddn.com/%E5%88%86%E5%8C%BA%E6%8C%82%E8%BD%BD%E6%83%85%E5%86%B5.png)
 可以看到/dev/sdb1已经被挂载到/backup目录下了。

### 交换分区的创建与挂载
交换分区的创建与挂载步骤与普通分区一致，也是先划分分区，然后在分区上创建文件系统，接着是激活交换分区。因为交换分区与普通分区有一点不同，需要先调整分区的分区ID。

*  将/dev/sdb2分区作为交换分区，首先需要修改/dev/sdb2分区的分区ID
```
[root@sg-pc /]# fdisk /dev/sdb

WARNING: DOS-compatible mode is deprecated. It's strongly recommended to
         switch off the mode (command 'c') and change display units to
         sectors (command 'u').

Command (m for help): p

Disk /dev/sdb: 21.5 GB, 21474836480 bytes
255 heads, 63 sectors/track, 2610 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x179a39d6

   Device Boot      Start         End      Blocks   Id  System
/dev/sdb1               1         654     5253223+  83  Linux
/dev/sdb2             655        1308     5253255   83  Linux
/dev/sdb3            1309        1962     5253255   83  Linux
/dev/sdb4            1963        2610     5205060    5  Extended
/dev/sdb5            1963        2224     2104483+  83  Linux

Command (m for help): l

 0  Empty           24  NEC DOS         81  Minix / old Lin bf  Solaris        
 1  FAT12           39  Plan 9          82  Linux swap / So c1  DRDOS/sec (FAT-
 2  XENIX root      3c  PartitionMagic  83  Linux           c4  DRDOS/sec (FAT-
 3  XENIX usr       40  Venix 80286     84  OS/2 hidden C:  c6  DRDOS/sec (FAT-
 4  FAT16 <32M      41  PPC PReP Boot   85  Linux extended  c7  Syrinx         
 5  Extended        42  SFS             86  NTFS volume set da  Non-FS data    
 6  FAT16           4d  QNX4.x          87  NTFS volume set db  CP/M / CTOS / .
 7  HPFS/NTFS       4e  QNX4.x 2nd part 88  Linux plaintext de  Dell Utility   
 8  AIX             4f  QNX4.x 3rd part 8e  Linux LVM       df  BootIt         
 9  AIX bootable    50  OnTrack DM      93  Amoeba          e1  DOS access     
 a  OS/2 Boot Manag 51  OnTrack DM6 Aux 94  Amoeba BBT      e3  DOS R/O        
 b  W95 FAT32       52  CP/M            9f  BSD/OS          e4  SpeedStor      
 c  W95 FAT32 (LBA) 53  OnTrack DM6 Aux a0  IBM Thinkpad hi eb  BeOS fs        
 e  W95 FAT16 (LBA) 54  OnTrackDM6      a5  FreeBSD         ee  GPT            
 f  W95 Ext'd (LBA) 55  EZ-Drive        a6  OpenBSD         ef  EFI (FAT-12/16/
10  OPUS            56  Golden Bow      a7  NeXTSTEP        f0  Linux/PA-RISC b
11  Hidden FAT12    5c  Priam Edisk     a8  Darwin UFS      f1  SpeedStor      
12  Compaq diagnost 61  SpeedStor       a9  NetBSD          f4  SpeedStor      
14  Hidden FAT16 <3 63  GNU HURD or Sys ab  Darwin boot     f2  DOS secondary  
16  Hidden FAT16    64  Novell Netware  af  HFS / HFS+      fb  VMware VMFS    
17  Hidden HPFS/NTF 65  Novell Netware  b7  BSDI fs         fc  VMware VMKCORE 
18  AST SmartSleep  70  DiskSecure Mult b8  BSDI swap       fd  Linux raid auto
1b  Hidden W95 FAT3 75  PC/IX           bb  Boot Wizard hid fe  LANstep        
1c  Hidden W95 FAT3 80  Old Minix       be  Solaris boot    ff  BBT            
1e  Hidden W95 FAT1

Command (m for help): t
Partition number (1-5): 2
Hex code (type L to list codes): 82
Changed system type of partition 2 to 82 (Linux swap / Solaris)

Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.

WARNING: Re-reading the partition table failed with error 16: Device or resource busy.
The kernel still uses the old table. The new table will be used at
the next reboot or after you run partprobe(8) or kpartx(8)
Syncing disks.
```
* 创建交换分区的文件系统
```
[root@sg-pc /]# mkswap -L 'myswap' /dev/sdb2
Setting up swapspace version 1, size = 5253248 KiB
LABEL=myswap, UUID=fee42b41-d38f-498e-a27f-70a962734a26
```
* 激活交换分区
```
[root@sg-pc /]# free -m
             total       used       free     shared    buffers     cached
Mem:           996        642        354          1        112        304
-/+ buffers/cache:        224        771
Swap:         2015          0       2015
[root@sg-pc /]# swapon -a /dev/sdb2
[root@sg-pc /]# free -m
             total       used       free     shared    buffers     cached
Mem:           996        645        350          1        112        304
-/+ buffers/cache:        228        767
Swap:         7146          0       7146
```

## 分区开机自动挂载
上述的硬盘的分区都是在系统中临时挂载使用的，当系统重启的话，目前我们挂载的分区并不会自动挂载上去，又需要我们执行挂载的步骤，这样很不方便。如果我们需要硬盘分区在系统重启的时候也能自动挂载，我们需要将分区的配置写到/etc/fstab这个文件中。
![](http://oc4wmeyj8.bkt.clouddn.com/%E8%87%AA%E5%8A%A8%E6%8C%82%E8%BD%BD1.png)
 /etc/fstab文件中每一行代表一个启动的时候需要自动挂载的硬盘分区。
每一行记录友6个字段组成，按照顺序分别为：
* 要挂载的设备
* 挂载点
* 文件系统类型
* 挂载选项
一般为defaults
* 转储频率
指的是备份频率，0表示不做备份，1表示每天做备份，2表示隔天做备份
* 自检次序
0表示不自检
1表示首先自检，数值越大自检次序越靠后。

### 示例
我们以上面划分的./dev/sdb1分区为例，将其设置为每次机器重启的时候都会将/dev/sdb1分区挂载到相应的挂载点。
![](http://oc4wmeyj8.bkt.clouddn.com/%E8%87%AA%E5%8A%A8%E6%8C%82%E8%BD%BD2.png)
 在/etc/fstab文件中添加如图所示的一行，表示每次机器重启的时候，都会将/dev/sdb1分区挂载到/backup挂载点上，并且该分区的文件系统类型为ext4类型，挂载选项是defautls，不备份，不自检。
## 总结
总的来说，Linux下的硬盘分区以及挂载主要是三个步骤：
1. 划分分区
2. 创建文件系统
3. 挂载分区