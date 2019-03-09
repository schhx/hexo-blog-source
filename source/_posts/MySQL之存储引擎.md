---
title: MySQL之存储引擎
categories:
  - MySQL
tags:
  - MySQL
date: 2018-09-22 08:58:41
---

存储引擎是什么？存储引擎说白了就是如何存储数据、如何为存储的数据建立索引和如何更新、查询数据等技术的实现方法。<!-- more -->我们对数据库的操作最终都是由存储引擎来执行，因此了解存储引擎对我们建立高效索引、优化数据库性能等有不小的帮助。

## 存储引擎的种类

MySQL支持的引擎很多，我们可以通过命令查看MySQL自带的引擎：



```
mysql> show engines;
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| Engine             | Support | Comment                                                        | Transactions | XA   | Savepoints |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| InnoDB             | DEFAULT | Supports transactions, row-level locking, and foreign keys     | YES          | YES  | YES        |
| MRG_MYISAM         | YES     | Collection of identical MyISAM tables                          | NO           | NO   | NO         |
| MEMORY             | YES     | Hash based, stored in memory, useful for temporary tables      | NO           | NO   | NO         |
| BLACKHOLE          | YES     | /dev/null storage engine (anything you write to it disappears) | NO           | NO   | NO         |
| MyISAM             | YES     | MyISAM storage engine                                          | NO           | NO   | NO         |
| CSV                | YES     | CSV storage engine                                             | NO           | NO   | NO         |
| ARCHIVE            | YES     | Archive storage engine                                         | NO           | NO   | NO         |
| PERFORMANCE_SCHEMA | YES     | Performance Schema                                             | NO           | NO   | NO         |
| FEDERATED          | NO      | Federated MySQL storage engine                                 | NULL         | NULL | NULL       |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
9 rows in set (0.00 sec)
```


上面就是我本机安装的MySQL(5.7.15)支持的引擎类型，下面我们会着重介绍其中常用的两款InnoDB和MyISAM，其余的会一笔带过。

同样我们也可以使用命令查看某一个数据库表的存储引擎：

```
mysql> show table status from db_name where name = 'user'\G
*************************** 1. row ***************************
           Name: user
         Engine: InnoDB
        Version: 10
     Row_format: Dynamic
           Rows: 2
 Avg_row_length: 8192
    Data_length: 16384
Max_data_length: 0
   Index_length: 0
      Data_free: 0
 Auto_increment: NULL
    Create_time: 2017-12-24 15:29:56
    Update_time: NULL
     Check_time: NULL
      Collation: utf8mb4_bin
       Checksum: NULL
 Create_options:
        Comment: 用户表
1 row in set (0.00 sec)
```

### InnoDB

InnoDB是MySQL默认的事务型引擎，也是最重要、使用最广泛的存储引擎，同时也是MySQL 5.2以及以后版本的默认引擎。InnoDB被设计用来处理大量的短期事务，但是由于它的性能和自动崩溃恢复的特性，使得它在非事务型存储的需求中也很流行。因此，除非有非常特殊的原因需要使用其他的存储引擎，否则应该优先考虑InnoDB引擎。

InnoDB将数据存储在表空间中，表空间是由InnoDB管理的一个黑盒子，由一系列数据文件组成，在MySQL 4.1 以后的版本中，InnoDB可以将每个表的数据和索引存放在单独的文件中。

InnoDB采用MVCC来支持高并发，并且实现了四个标准的隔离级别(未提交读，提交读，可重复读，可串行化)，其默认级别是可重复读，并且通过间隙锁策略防止幻读的出现。

InnoDB表是基于聚簇索引建立的，聚簇索引对主键查询有很高的性能，不过它的非主键索引中必须包含主键列，所以如果主键列很大的话，其他的所有索引都会很大。


### MyISAM

在MySQL 5.1以及以前的版本中，MyISAM是默认的存储引擎，它提供了大量的特性，包括全文索引、压缩空间函数等，但MyISAM不支持事务和行级锁，而且崩溃后无法安全恢复。

MyISAM会将表存储在两个文件中：数据文件和索引文件，分别以.MYD和.MYI为扩展名。MyISAM的表定义存储在.frm文件中。

MyISAM加锁是针对整张表，而不是准对行。读取时会对需要读到的所有表加共享锁，写对则对表加排它锁，但是在表有读取查询的同时，也可以并发插入。

对于MyISAM表，即使是BLOB和TEXT等长字段，也可以基于其前500字符创建索引。MyISAM支持全文索引，可以支持复杂的查询。另外MyISAM支持延迟更新索引。

如果表在创建并导入数据以后，不会再进行修改操作，这样的表或许适合采用MyISAM压缩表。压缩表可以极大地减少磁盘空间占用，因此可以减少磁盘I/O，从而提升查询性能。


### Archive

Archive引擎只支持insert和select操作，Archive会缓存所有的写并对插入的行进行压缩，所以比MyISAM表的磁盘I/O更少。但是每次select查询都需要执行全表扫描，所以Archive更适合全表扫描的应用，比如日志分析或者数据采集类应用。

### Blackhole

Blackhole引擎没有实现任何的存储机制，他会丢弃所有插入的数据，不做任何保存服务器会记录Blackhole表的日志，所以可以用于复制数据到备库。或者只是简单地记录到日志。

### CSV

CSV引擎可以将普通的CSV文件作为MySQL的表来处理，但是这种表不支持索引。CSV引擎可以加载CSV文件处理，也可以把数据导出到CSV文件，因此CSV引擎可以作为一种数据交换的机制。

### Memory

Memory引擎将所有的数据都保存在内存中，所以它的查询速度非常快，但是Memory表在重启后会丢失数据，但是表结构还会保留。Memory表是表级锁，因此并发写入的性能比较低，所以Memory表适合需要快速访问，数据不怎么修改，而且重启丢失也没关系的场景，比如查找或映射表，保存数据分析的中间数据等。









