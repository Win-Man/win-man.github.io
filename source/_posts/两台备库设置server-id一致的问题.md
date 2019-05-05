---
title: 两台备库设置server_id一致的问题
date: 2017-07-11 08:13:04
tags:
- MySQL
- 复制
categories:
- MySQL
---
## 简介
在《高性能MySQL》上看到说在一主多从的架构中，如果从库的sever_id设置为一样的，可能会导致一些奇怪的现象，例如从库的错误日志中会不断的打印错误日志，会不断的断开连接并重新连接。

>在主库上，会发现两台备库只有一台连接到主库（通常情况下所有的备库都会建立连接等待随时进行复制）。在备库的错误日志中，则会发现反复的重连和连接断开信息，但不会提及被错误配置的服务器ID。

根据眼见为实，耳听为虚的原则，我搭了一套一主两从的环境，将两台从库的server_id设置为了一样的。但是并没有看到书上提到的现象。考虑了一下，会不会是数据库版本的原因，因为我的测试的数据库版本是5.7.18版本的。于是又重新搭建了一套5.5.36版本的数据库，并成功复现了书上所讲的现象。
下面是我的测试步骤。

## 测试步骤

### 环境介绍
* 主库
IP：192.168.1.130
server_id：3656
* 从库A
IP：192.168.1.36
server_id：56
* 从库B
IP：192.168.1.57
server_id：56

三台主机除server_id之外，其余配置如下：
```
server_id = 123
[client]
socket = /home/mysql/data/mysqldata5.5/sock/mysql.sock
[mysqld]
#server_id = 3655
server_id = 123
port = 3306
skip_name_resolve = 1
binlog_format = ROW
#binlog_format = STATEMENT
basedir = /home/mysql/program/mysql5.5.36
datadir = /home/mysql/data/mysqldata5.5/mydata
socket = /home/mysql/data/mysqldata5.5/sock/mysql.sock
pid-file = /home/mysql/data/mysqldata5.5/sock/mysql.pid
tmpdir = /home/mysql/data/mysqldata5.5/tmpdir
log-error = /home/mysql/data/mysqldata5.5/log/error.log
slow_query_log
slow_query_log_file = /home/mysql/data/mysqldata5.5/slowlog/slow-query.log
log-bin = /home/mysql/data/mysqldata5.5/binlog/mysql-bin
relay-log = /home/mysql/data/mysqldata5.5/relaylog/mysql-relay-bin
innodb_data_home_dir = /home/mysql/data/mysqldata5.5/innodb_ts
innodb_log_group_home_dir = /home/mysql/data/mysqldata5.5/innodb_log
#innodb_undo_directory = /home/mysql/data/mysqldata5.5/undo/
sync_binlog=1
innodb_file_per_table=1
#skip_grant_tables
expire_logs_days = 1
log_slave_updates = ON
#replicate-same-server-id=1
skip_slave_start
#innodb_undo_tablespaces=1
```
 
### 5.5.36版本现象
初始搭建环境之后，查看各主机状态。搭建环境的步骤就省略。
#### 主库（192.168.1.130）
主库通过show processlist语句查看，只有一个dump线程，但是通过多次刷新，可以看到连接的是不同的服务器。可以看到每次通过show processlist语句显示的dump线程的Host字段中，IP:PORT的值是不断在更新的，说明dump线程在不断的重连，才会出现占用不同的端口的现象。

 
#### 从库A（192.168.1.36）
通过`show slave status\G`命令查看复制状态，多次执行可以看到`Slave_IO_Running`字段显示的内容，出现YES或者Connnecting两种状态。可以看到I/O线程在不断的进行重连。
并且通过`tail -f `命令查看error log，可以看到I/O线程一直在尝试重新连接。

 
 可以看到在错误日志中打印的信息是，I/O线程连接

 
#### 从库B（192.168.1.57）
从库B现象与从库A一致。

 


 
### 5.6.36版本现象
搭建环境步骤省略。

#### 主库（192.168.1.130）
show processlist查看有两个dump线程，并且多次刷新，发现Host字段中的IP:PORT并没有修改，说明dump线程一直保持连接。


#### 从库A（192.168.1.36）
tail -f /home/mysql/data/mysqldata5.6/log/error.log查看错误日志，没有不断断开连接

 
#### 从库B（192.168.1.57）
tail -f /home/mysql/data/mysqldata5.6/log/error.log查看错误日志，没有不断断开连接

 
## 原因分析
[http://www.penglixun.com/tech/database/mysql_multi_slave_same_serverid.html](http://www.penglixun.com/tech/database/mysql_multi_slave_same_serverid.html)这是彭大大写的关于多个slave使用相同server_id时冲突的原因。按照彭大大的分析，我理解的是，slave的I/O线程连接上主库的时候，主库上会调用`register_slave()`这个函数，在这个函数中又调用了unregister_slave()函数，会将之前使用相同server_id的线程给注销掉。从而导致从库的I/O线程不断断开重连。
但是仔细看了一下unregister_slave()函数的代码，并没有发现MySQL是根据server_id来注销dump线程的。并且进一步比较了一下5.5.36和5.6.36版本的代码，并没有发现不同。
进一步看了一下彭大大的文章，发现有人在下面评论，说主要是`kill_zombie_slave_threads()`函数导致的。于是看了一下`kill_zombie_slave_threads()`函数的逻辑，发现MySQL应该就是在这一步根据server_id将线程kill了。

* 5.5.36版本
首先来看下5.5.36版本的`kill_zombie_dump_threads()`函数的代码。看到这个函数传入的参数是一个uint32类型的slave_server_id,在函数中做的事情是，遍历MySQL中的所有线程，如果遍历到一个线程是dump线程并且线程的server_id是等于传入的参数值话，则跳出遍历循环，并对kill掉这个线程。
```
void kill_zombie_dump_threads(uint32 slave_server_id)                           
{                                                                               
  mysql_mutex_lock(&LOCK_thread_count);                                         
  I_List_iterator<THD> it(threads);                                             
  THD *tmp;                                                                     
  while ((tmp=it++))                                                            
  {                                                                             
    if (tmp->command == COM_BINLOG_DUMP &&                                      
      tmp->server_id == slave_server_id)                                       
    {                                                                           
     mysql_mutex_lock(&tmp->LOCK_thd_data);    // Lock from delete             
     break;                                                                    
    }                                                                           
  }                                                                             
  mysql_mutex_unlock(&LOCK_thread_count);                                       
  if (tmp)                                                                      
  {                                                                             
    /*                                                                          
     Here we do not call kill_one_thread() as                                  
     it will be slow because it will iterate through the list                  
     again. We just to do kill the thread ourselves.                           
    */                                                                          
    tmp->awake(THD::KILL_QUERY);                                                
    mysql_mutex_unlock(&tmp->LOCK_thd_data);                                    
  }                                                                             
} 
```

* 5.6.35版本
再来看一下5.6.36版本的`kill_zombie_dump_threads()`函数的代码实现，与5.5.36大不相同。首先传入的参数是一THD类型的指针，在函数中实现的逻辑同样是遍历MySQL中的所有线程，如果找到dump线程，首先看一下这个线程有没有uuid字段（因为uuid是在5.6之后的版本才有的，这边是为了兼容5.5），如果有uuid则用uuid进行比较，如果没有uuid，则用server_id进行比较。
```
 void kill_zombie_dump_threads(THD *thd)                                                                                                  
{                                                                               
  String slave_uuid;                                                            
  get_slave_uuid(thd, &slave_uuid);                                             
  if (slave_uuid.length() == 0 && thd->server_id == 0)                          
    return;                                                                     
                                                                                
  mysql_mutex_lock(&LOCK_thread_count);                                         
  THD *tmp= NULL;                                                               
  Thread_iterator it= global_thread_list_begin();                               
  Thread_iterator end= global_thread_list_end();                                
  bool is_zombie_thread= false;                                                 
  for (; it != end; ++it)                                                       
  {                                                                             
    if ((*it) != thd && ((*it)->get_command() == COM_BINLOG_DUMP || (*it)->get_command() == COM_BINLOG_DUMP_GTID))         
    {                                                                           
     String tmp_uuid;                                                          
     get_slave_uuid((*it), &tmp_uuid);                                         
     if (slave_uuid.length())                                                  
     {                                                                         
       is_zombie_thread= (tmp_uuid.length() && !strncmp(slave_uuid.c_ptr(),                         
                                         tmp_uuid.c_ptr(), UUID_LENGTH)); 
                                                                       
     else                                                                      
     {                                                                         
       /*                                                                      
       ¦ Check if it is a 5.5 slave's dump thread i.e., server_id should be    
       ¦ same && dump thread should not contain 'UUID'.                        
       */                                                                      
       is_zombie_thread= (((*it)->server_id == thd->server_id) && !tmp_uuid.length());                                 
     }                                                                         
     if (is_zombie_thread)                                                     
     {                                                                         
       tmp= *it;                                                               
       mysql_mutex_lock(&tmp->LOCK_thd_data);  // Lock from delete             
       break;                                                                  
     }                                                                         
    }                                                                           
  }                                                                             
  mysql_mutex_unlock(&LOCK_thread_count);                                       
  if (tmp)                                                                      
  {                                                                             
    /*                                                                          
    ¦ Here we do not call kill_one_thread() as                                  
    ¦ it will be slow because it will iterate through the list                  
    ¦ again. We just to do kill the thread ourselves.                           
    */                                                                          
    if (log_warnings > 1)                                                       
    {                                                                           
     if (slave_uuid.length())                                                  
     {                                                                       
 sql_print_information("While initializing dump thread for slave with "  
"UUID <%s>, found a zombie dump thread with the " 
 "same UUID. Master is killing the zombie dump "   
 "thread(%lu).", slave_uuid.c_ptr(),               
 tmp->thread_id);                                  
     }                                                                         
     else                                                                      
     {                                                                         
       sql_print_information("While initializing dump thread for slave with "  
 "server_id <%u>, found a zombie dump thread with the "
"same server_id. Master is killing the zombie dump "
 "thread(%lu).", thd->server_id,                   
 tmp->thread_id);                                  
     }                                                                         
    }                                                                           
    tmp->duplicate_slave_id= true;                                              
    tmp->awake(THD::KILL_QUERY);                                                
    mysql_mutex_unlock(&tmp->LOCK_thd_data);                                    
  }                                                                             
}                                                  
```
* 函数调用
知道了`kill_zombie_dump_threads()`线程实现的逻辑，那MySQL是在什么地方会调用这个函数的呢。看了一下函数是在`case COM_BINLOG_DUMP`中被调用的。
在5.5.36版本中是在
```
case COM_BINLOG_DUMP:                                                         
 {                                                                           
 ulong pos;                                                                
 ushort flags;                                                             
 uint32 slave_server_id;                                                                                                                                 
 status_var_increment(thd->status_var.com_other);                          
 thd->enable_slow_log= opt_log_slow_admin_statements;                      
 if (check_global_access(thd, REPL_SLAVE_ACL))                             
break;                                                                                                                                                 
 /* TODO: The following has to be changed to an 8 byte integer */          
 pos = uint4korr(packet);                                                  
 flags = uint2korr(packet + 4);                                            
 thd->server_id=0; /* avoid suicide */                                     
 if ((slave_server_id= uint4korr(packet+6))) // mysqlbinlog.server_id==0   
 kill_zombie_dump_threads(slave_server_id);                                  
 thd->server_id = slave_server_id;                                                                                                                    
 general_log_print(thd, command, "Log: '%s'  Pos: %ld", packet+10,  (long) pos);                                              
 mysql_binlog_send(thd, thd->strdup(packet + 10), (my_off_t) pos, flags);  
unregister_slave(thd,1,1);                                                
/*  fake COM_QUIT -- if we get here, the thread needs to terminate */     
 error = TRUE;                                                             
break;                                                                    
}    
```
在5.6.36版本中也是在`case COM_BINLOG_DUMP`中，只不过是将之前的逻辑封装在了`com_binlog_dump()`函数中了，`kill_zombie_dump_threads()`也是在`com_binlog_dump()`函数中调用的。
```
case COM_BINLOG_DUMP:                                                         
error= com_binlog_dump(thd, packet, packet_length);                                                                                  
break;    
```
`case COM_BINLOG_DUMP`中所进行的操作就是将dump线程通知I/O线程拉取新的binlog。

## 总结
整理下来的话，基本上可以确定主要是因为`kill_zombie_dump_threads()`函数导致在5.6之前的版本中，如果是一主多从的架构中，如果在从库之间的server_id如果设置为一样，会出现从开I/O线程不断断开重连的现象。因为在5.6之前的版本中，还没有UUID的概念，MySQL使用server_id来区分是否是同一台机器，而在5.6之后的版本是使用的UUID来区分。
总结一句，就是数据库之间的server_id不要设置成一样，不然可能会有一些不可预知的错误。

> 博客地址：https://win-man.github.io/  
> 公众号：欢迎关注  

![](https://user-gold-cdn.xitu.io/2018/8/16/165435ce71d2b88b?w=258&h=258&f=jpeg&s=26568)
