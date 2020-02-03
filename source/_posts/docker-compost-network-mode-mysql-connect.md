---
title: docker-compose 中 network_mode 设置导致无法从容器外部访问 MySQL
date: 2020-02-03 21:35:13
tags:
- MySQL
- Docker
categories:
- Docker
---
# docker-compose 中 network_mode 设置导致无法从容器外部访问 MySQL
## 问题现象
通过 docker-compose 在自己电脑上部署 MySQL，启动之后，查看容器状态正常，进入容器内部访问 MySQL 正常，但是在宿主机上连接 MySQL 失败。

* docker-compose.yml 配置文件

```
$ cat docker-compose.yml
version: '2'
services:
    mysql:
        network_mode: "host"
        environment:
            MYSQL_ROOT_PASSWORD: "xxxx"
            MYSQL_USER: "qbench"
            MYSQL_PASS: "xxxx"
        image: "docker.io/mysql:5.7.22"
        ports:
            - "3306:3306"
        restart: always
        volumes:
            - "./db:/var/lib/mysql"
            - "./conf/my.cnf:/etc/my,cnf"
            - "./init:/docker-entrypoint-initdb.d/"
```

* 拉取镜像

```
$ docker-compose -f docker-compose.yml pull
```

* 启动

```
$ docker-compost -f docker-compose.yml up -d
```

* 从宿主机上连接 MySQL 报错

```
[20-02-03 19:46:34]  shengang@abcs-MacBook-Pro  ~/Documents/002-workspace/docker-workspace/mysql-5.7.22
$ docker container ls
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
4865e2c56f67        mysql:5.7.22        "docker-entrypoint.s…"   6 seconds ago       Up 5 seconds                            mysql-5722_mysql_1
[20-02-03 19:46:39]  shengang@abcs-MacBook-Pro  ~/Documents/002-workspace/docker-workspace/mysql-5.7.22
$ mysql -uroot -P3306 -h127.0.0.1 -p
Enter password:
ERROR 2003 (HY000): Can't connect to MySQL server on '127.0.0.1' (61)
```

* 但是进入容器内部连接 MySQL 正常

```
[20-02-03 19:46:56]  shengang@abcs-MacBook-Pro  ~/Documents/002-workspace/docker-workspace/mysql-5.7.22
$ docker exec -it 4865e2c56f67 bash
root@linuxkit-025000000001:/# mysql -uroot -P3306 -h127.0.0.1 -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.22 MySQL Community Server (GPL)

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```

## 问题排查

排查过程其实比较简单，因为从容器内部连接正常，从宿主机连接失败，这个基本上就是网络或者端口或者防火墙的问题了。首先看 `docker container ls` 的结果输出中 `PORTS` 列的内容是空的，说明端口没有映射出来。但是在 docker-compose.yml 配置文件中我们明明设置了 ports 配置。仔细看了下 docker-compose.yml 配置文件，发现一个 `network_mode: "host"` 的配置项，这个我之前没怎么了解过，只是抄的网上的配置文件。大概率就是这个配置引起的问题。于是尝试了一下将 `network_mode: "host"` 这一行删除掉了。然后通过 `docker-compose -f docker-compose.yml down` ,`docker-compose -f docker-compose.yml up -d` 重新部署容器，连接就正常了。

## 问题原因
虽然问题解决了，但是最终还是需要了解一下是为什么会导致问题的出现，得好好理解一下 network_mode 这个配置的含义。

docker-compose.yml 配置文件中的 netwokr_mode 是用于设置网络模式的，与 docker run 中的 --network 选项参数一样，加上了 `service:[service name]` 形式的配置，默认是 bridge 桥接模式

* network_mode: "bridge"

默认的网络模式。如果没有指定网络驱动，默认会创建一个 bridge 类型的网络。桥接模式一般是用在应用是独立的情况，容器之间不需要互相通信。

* network_mode: "host"

host 网络模式，针对的也是应用是独立的情况，在 host 网络模式下，移除了宿主机与容器之间的网络隔离，荣容器直接使用宿主机的网络，这样就能在容器中访问宿主机网络。host 网络模式只对 Docker 17.06 以及更高版本的 swarm 服务可用。

* network_mode: "none"

none 表示对于这个 container ，禁用所有的网络。


* network_mode: "service:[service name]"

* network_mode: "container:[container name/id]"

当在 swarm 服务中，network_mode 选项会被忽略。

