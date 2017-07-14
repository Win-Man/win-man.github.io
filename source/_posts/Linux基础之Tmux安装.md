---
title: Linux基础之Tmux安装
date: 2016-12-28 20:02:11
tags:
- Linux
- 工具
categories:
- Linux
---
## 1.前言

　　Linux 命令行装逼利器——Tmux。转行 DBA 之后整天和命令行打交道，在装逼的道路上越走越远了。Tmux 是 Linux 命令行下一个工具，让我可以更加方便的**装逼**。先讲一下怎么安装这个工具，之后再写一篇文章记录一下怎么使用这个工具。

**系统环境：**

> CentOS 6.5

## 2.安装过程

　　`tmux` 工具依赖于 `libevent` 和 `ncurses` 两个软件包。我们先安装这两个软件包。

### 2.1 libevent 安装

``` c
// 下载 tar.gz 文件  
// 可以在 http://libevent.org/ 网站上找到最新版本
$ wget https://cloud.github.com/downloads/libevent/libevent/libevent-2.0.21-stable.tar.gz
// 解压 tar.gz 文件
$ tar zxvf libevent-2.0.21-stable.tar.gz
$ cd libevent-2.0.21-stable
// 配置安装路径并编译安装，这边可能会出现127错误，可以在./configure中加上--without-cxx-binding参数
$ ./configure --prefix=/usr/local/libevent
$ make
$ make install
```

### 2.2 ncurses 安装

``` c
$ yum install ncurses-devel
```

### 2.3 tmux 安装

``` c
// 下载 tar.gz 文件 在 https://tmux.github.io/ 可以下载到最新版
$ wget https://github.com/tmux/tmux/releases/download/2.3/tmux-2.3.tar.gz
// 解压 tar.gz 文件
$ tar zxvf tmux-2.3.tar.gz 
$ cd tmux-2.3
// 配置安装参数并编译安装
$ CFLAGS="-I/usr/local/libevent/include -I/usr/local/ncurses/include" LDFLAGS="-L/usr/local/libevent/lib -L/usr/local/ncurses/lib" ./configure --prefix=/usr/local/tmux
$ make
$ make install
```
### 2.4 配置 PATH 环境变量

在 /etc/profile 最末尾增加 `PATH=$PATH:/usr/local/tmux/bin`，并使用 `source` 命令生效。

``` c
// 将 tmux 拷贝到环境变量目录下
$ cp /usr/local/tmux/bin/tmux /usr/local/bin/
```

## 3.错误及解决方案

### 3.1 问题一

　　运行 `tmux` 命令提示 `libevent-2.0.so.5` 找不到。

![](http://oc4wmeyj8.bkt.clouddn.com/tmux%E5%AE%89%E8%A3%85%E9%97%AE%E9%A2%981.png)

　　**解决方案**：
``` c
// 将 libevent-2.0.so.5 移动到 /usr/lib64 下（64位操作系统） | /usr/lib 下（32位操作系统）
$ cp /usr/local/libevent/lib/libevent-2.0.so.5 /usr/lib64
```