---
layout:     post
title:      "MySQL存储引擎－MyISAM与InnoDB区别"
subtitle:   ""
date:       2018-09-05
author:     "Liuz2015"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - 数据库
    - MySQL
    - InnoDB
    - MyISAM
---

## 目录
- [简介](#简介) 
- [区别](#区别)
- [总结](#总结)
- [参考资料](#参考资料)

## 简介

MyISAM是MySQL的一种数据库引擎，在早期的ISAM（Indexed Sequential Access Method：有索引的顺序访问方法）上进行改良。虽然性能极佳，但却有一个缺点：不支持事务处理（transaction）。

在这几年的发展下，MySQL也导入了InnoDB（另一种数据库引擎），以强化参考完整性与并发违规处理机制，后来就逐渐取代MyISAM。

InnoDB，为MySQL AB发布binary的标准之一。InnoDB由Innobase Oy公司所开发，2006年五月时由甲骨文公司并购。与传统的ISAM与MyISAM相比，InnoDB的最大特色就是支持了ACID兼容的事务（Transaction）功能。目前InnoDB采用双轨制授权，一是GPL授权，另一是专有软件授权。

那么MyISAM与InnoDB的区别是什么呢？

## 区别

### 1、存储结构

MyISAM：每个MyISAM在磁盘上存储成三个文件。第一个文件的名字以表的名字开始，扩展名指出文件类型。.frm文件存储表定义。数据文件的扩展名为.MYD (MYData)。索引文件的扩展名是.MYI (MYIndex)。

InnoDB：所有的表都保存在同一个数据文件中（也可能是多个文件，或者是独立的表空间文件），InnoDB表的大小只受限于操作系统文件的大小，一般为2GB。

### 2、 存储空间

MyISAM：可被压缩，存储空间较小。支持三种不同的存储格式：静态表(默认，但是注意数据末尾不能有空格，会被去掉)、动态表、压缩表。

InnoDB：需要更多的内存和存储，它会在主内存中建立其专用的缓冲池用于高速缓冲数据和索引。

### 3、 可移植性、备份及恢复

MyISAM：数据是以文件的形式存储，所以在跨平台的数据转移中会很方便。在备份和恢复时可单独针对某个表进行操作。

InnoDB：免费的方案可以是拷贝数据文件、备份 binlog，或者用 mysqldump，在数据量达到几十G的时候就相对痛苦了。

### 4、 事务支持

MyISAM：强调的是性能，每次查询具有原子性,其执行数度比InnoDB类型更快，但是不提供事务支持。

InnoDB：提供事务支持事务，外部键等高级数据库功能。 具有事务(commit)、回滚(rollback)和崩溃修复能力(crash recovery capabilities)的事务安全(transaction-safe (ACID compliant))型表。

### 5、 AUTO_INCREMENT

MyISAM：可以和其他字段一起建立联合索引。引擎的自动增长列必须是索引，如果是组合索引，自动增长可以不是第一列，他可以根据前面几列进行排序后递增。

InnoDB：InnoDB中必须包含只有该字段的索引。引擎的自动增长列必须是索引，如果是组合索引也必须是组合索引的第一列。

### 6、 表锁差异

MyISAM：只支持表级锁，用户在操作myisam表时，select，update，delete，insert语句都会给表自动加锁，如果加锁以后的表满足insert并发的情况下，可以在表的尾部插入新的数据。也可以通过lock table命令来锁表，这样操作主要是可以模仿事务，但是消耗非常大。

InnoDB：支持事务和行级锁，是innodb的最大特色。行锁大幅度提高了多用户并发操作的新能。但是InnoDB的行锁，只是在WHERE的主键是有效的，非主键的WHERE都会锁全表的。查看mysql的默认事务隔离级别：
“show global variables like ‘tx_isolation’; ”
Innodb的行锁模式有以下几种：共享锁，排他锁，意向共享锁(表锁)，意向排他锁(表锁)，间隙锁。

关于死锁：当两个事务都需要获得对方持有的排他锁才能完成事务，这样就导致了循环锁等待，也就是常见的死锁类型。
解决死锁的方法：
- 数据库参数
- 应用中尽量约定程序读取表的顺序一样
- 应用中处理一个表时，尽量对处理的顺序排序
- 调整事务隔离级别（避免两个事务同时操作一行不存在的数据，容易发生死锁）

### 7、 索引

#### 自动增长

MyISAM：引擎的自动增长列必须是索引，如果是组合索引，自动增长可以不是第一列，他可以根据前面几列进行排序后递增。

InnoDB：引擎的自动增长列必须是索引，如果是组合索引，也必须是组合索引的第一列。

#### 全文索引

MyISAM：支持 FULLTEXT类型的全文索引。

InnoDB：不支持FULLTEXT类型的全文索引，但是innodb可以使用sphinx插件支持全文索引，并且效果更好。

#### 索引保存位置

MyISAM：索引以表名+.MYI文件分别保存。

InnoDB：索引和数据一起保存在表空间里。

### 8、 表主键

MyISAM：允许没有任何索引和主键的表存在，索引都是保存行的地址。

InnoDB：如果没有设定主键或者非空唯一索引，就会自动生成一个6字节的主键(用户不可见)，数据是主索引的一部分，附加索引保存的是主索引的值。

### 9、 表的具体行数

MyISAM：保存有表的总行数，如果select count(*) from table;会直接取出出该值。

InnoDB：没有保存表的总行数，如果使用select count(*) from table；就会遍历整个表，消耗相当大，但是在加了wehre条件后，myisam和innodb处理的方式都一样。

### 10、 CURD操作

MyISAM：如果执行大量的SELECT，MyISAM是更好的选择。

InnoDB：如果你的数据执行大量的INSERT或UPDATE，出于性能方面的考虑，应该使用InnoDB表。DELETE 从性能上InnoDB更优，但DELETE FROM table时，InnoDB不会重新建立表，而是一行一行的删除，在innodb上如果要清空保存有大量数据的表，最好使用truncate table这个命令。

### 11、 外键

MyISAM：不支持

InnoDB：支持


## 总结

1、MyISAM：基于传统的ISAM类型。不是事务安全的，而且不支持外键，如果执行大量的select、insert，MyISAM比较适合。

2、InnoDB：支持事务安全的引擎，支持外键、行锁、事务是他的最大特点。如果有大量的update和insert，建议使用InnoDB，特别是针对多个并发和QPS较高的情况。

通过上述的分析，基本上可以考虑使用InnoDB来替代MyISAM引擎了，原因是InnoDB自身很多良好的特点，比如事务支持、存储 过程、视图、行级锁定等等，在并发很多的情况下，相信InnoDB的表现肯定要比MyISAM强很多。另外，任何一种表都不是万能的，只用恰当的针对业务类型来选择合适的表类型，才能最大的发挥MySQL的性能优势。如果不是很复杂的Web应用，非关键应用，还是可以继续考虑MyISAM的，这个具体情况可以自己斟酌。

## 参考资料
- [MySQL引擎MyISAM和InnoDB区别详解](https://www.linuxidc.com/wap.aspx?nid=152040)

