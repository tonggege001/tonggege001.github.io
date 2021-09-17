---
layout: post  
title: 2021-09-15-MySQL技术内幕InnoDB存储引擎
date: 2021-09-15
categories: blog
tags: [数据库,技术]
description: MySQL技术内幕InnoDB存储引擎，记录一下关键的笔记内容。
---  

## 1-MySQL 体系结构和存储引擎  
MySQL被设计为一个单进程多线程架构的数据库，与SQL Server类似，也就是说MySQL数据库实例在系统上的表现就是一个进程。  

查看MySQL后台进程情况：  
```shell  
ps -ef | grep mysqld
```  

MySQL由以下几部分组成：  
连接池组件、管理服务和工具组件、SQL接口组件、查询分析器组件、优化器组件、缓冲(Cache)组件、插件式存储引擎、物理文件。  
需要注意的是，存储引擎是基于表的，而不是数据库。存储引擎是MySQL区别于其他数据库的一个最重要的特性，其好处是每个存储引擎都有各自的特点，能够根据具体应用建立不同存储引擎表，MySQL的核心在于存储引擎。由于MySQL开源特性，用户可以根据MySQL预定义的存储引擎借口编写自己的存储引擎，若用户对某一种存储引擎的性能或功能不满意，可以通过修改源码来得到想要的特性。  

**InnoDB存储引擎**  
支持事务，其特点是行锁设计、支持外键，并支持非锁定读，即默认读取操作不会产生锁。InnoDB存储引擎将数据房子啊一个逻辑的表空间，由其进行管理。InnoDB引擎通过使用多版本并发控制MVCC来获得高并发性，并且实现了SQL标准的4种隔离级别，默认为REPEATABLE级别，同时，使用一种被称为next-key locking的策略来避免幻读(phantom)现象的产生。除此之外，InnoDB存储引擎还提供了插入缓冲、二次写、自适应哈希索引、预读等高性能和高可用的功能。 数据采用聚类方式，因此每张表的存储都是按主键的顺序进行存放。  

**MyISAM存储引擎**
MyISAM存储引擎不支持事务、表锁设计，支持全文索引。MyISAM只缓存索引不缓存数据。M与ISAM存储引擎表由MYD和MYI组成，MYD用来存放数据文件，MYI用来存放索引文件。  

**NDB存储引擎**  
NDB存储引擎是一个集群存储引擎，其结构使share nothing的集群架构，因此能提供更高的可用性。NDB的特点是数据全部放在内存（从MySQL5.1开始，也可以将非索引数据存放在磁盘上），因此主键查找非常快，并且通过添加NDB数据存储节点可以线性地提高数据库性能。但是NDB连接操作(JOIN)是在MySQL数据库层完成的，而不是在存储引擎层完成的。这意味着复杂的连接操作需要巨大的网络开销，因此查询速度会变慢。  

**Memory存储引擎**  
Memory存储引擎将表中的数据存放在内存中，如果数据库重启或发生故障崩溃，表中的数据则会消失。它非常适合存储临时数据的临时表，以及数据仓库中的纬度表。Memory引擎默认使用哈希索引，而不是我们熟知的B+树索引。Memory引擎只支持表锁，并发性能较差，并且不支持TEXT和BLOB数据类型。最重要的是，存储变长字段Varchar时是按照定长字段char的方式进行的，因此会浪费内存。若MySQL使用Memory引擎，则当查询的临时表的中间结果集大于Memory引擎设置的容量则MySQL数据库会把其转换到MyISAM存储引擎表而存放到磁盘中，因此性能会有损失。

**Archive存储引擎**
只支持INSERT和SELECT操作，Archive存储引擎使用zlib算法将数据行进行压缩，压缩比一般可达1:10。它一般用于归档数据，如日志信息。Archive引擎使用行锁来实现高并发插入操作，但是其本身并不是事物安全的存储引擎，其设计目标主要是提供高速的插入和压缩功能。  

查看MySQL支持的引擎：  
```shell  
SHOW ENGINES 
```  

### 连接MySQL  
连接MySQL本质上就是进程间通信，进程间通信方式有管道、TCP/IP套接字、共享内存和消息队列、UNIX域套接字。  

#### TCP/IP  
例如：  
```shell  
mysql -h192.168.0.101 -u david -p
```
当连接时，MySQL先要校验权限，也就是检查一张权限视图用来判断IP是否允许连接。查看权限视图：  
```shell  
mysql>USE mysql
Database changed
mysql>SELECT host,user,password FROM user;
```
结果显示：  
![](/images/20210915-1.png)  

#### 命名管道和共享内存  
在MySQL配置文件中启用--enable-named-pipe选项可以使用命名管道连接MySQL。如果想使用共享内存方式，在连接时还必须使用--protocol=memory选项。  

#### UNIX域套接字  
Linux和UNIX环境下，还可以使用UNIX域套接字。UNIX域套接字不是一个网络协议，所以只能在MySQL客户端和数据库实例在一台服务器上的情况下使用。用户可以在配置文件中指定套接字文件的路径，如--socket=/tmp/mysql.sock。当数据库实例启动后，用户可以通过下列命令来进行UNIX域套接字文件的查找：  
```shell  
mysql>SHOW VARIABLE LIKE 'socket'
```  

在知道了UNIX域套接字文件的路径后，就可以使用该方式进行连接了，如下所示：  
```shell  
mysql -udavid -S /tmp/mysql.sock
```

## 2-InnoDB存储引擎  
### InnoDB体系架构  
InnoDB存储引擎有多个内存块，可以认为这些内存块组成了一个大的内存池。  
![](/images/20210915-3.png)
后台线程主要作用是负责刷新内存池中的数据，保证缓冲池中的内存缓存的是最近的数据。此外将已修改的数据文件刷新到磁盘文件，同时保证在数据库发生异常的情况下InnoDB能恢复到正常运行状态。  

### 后台线程  
**Master Thread**  
主要负责将缓冲池中的数据异步刷新到磁盘，保证数据一致性，包括*脏页的刷新*，*合并插入缓冲*，*UNDO页的回收*等。  

**IO Thread**  
在InnoDB存储引擎里大量使用了AIO(Aysnc IO)来处理写IO请求，这样可以极大提高数据库性能，而IO Thread的工作主要是负责这些IO请求的回调。有四种分别是：write、read、insert buffer、log。

**Purge Thread**  
事务被提交后，其所使用的undolog可能不再需要，因此需要PurgeThread来回收已经使用并分配的undo页。  

**Page Cleaner Thread**
将脏页的刷新操作独立出来处理，减轻master thread的压力，以及对用户查询线程的阻塞。  

### 内存  
**缓冲池**
InnoDB存储引擎是基于磁盘存储的，并将其中的记录按照页的方式进行管理。因为CPU速度和磁盘速度差异较大，因此使用缓冲池技术来提高数据库的整体性能。缓冲池实际上就是内存区域，在数据库中进行读取页的操作，首先讲磁盘读到的页存放在缓冲池中，这个过程称为将页FIX在缓冲池中。下一次读取相同的页时，首先判断该页是否在缓冲池中。若在缓冲池中，则称该页在缓冲池被命中，否则读取磁盘。  

对数据库中页的修改操作，则首先修改缓冲池的页，然后以一定频率刷新到磁盘上。页从缓冲池刷新回磁盘的操作并不是在每次页发生更新时出发，而是通过一种称为Checkpoint的机制刷新回磁盘。

查看缓冲池参数：  
```shell  
mysql>SHOW VARIABLES LIKE 'innodb_buffer_pool_size'
```  
缓冲池里存放索引页、数据页、undo页、插入缓冲、自适应索引、InnoDB存储的锁信息、数据字典信息，使用**LRU**(Latest Recent Used，最近最少使用)算法进行管理。即最频繁使用的页在LRU列表的前端，而最少使用的页在LRU列表的尾端，当缓冲池不能存放新的页时，首先释放LRU列表尾端的页。  

InnoDB对LRU算法做了改进，LRU列表中加入了midpoint位置。新读取到的页，虽然是最新访问的页，但并不是直接放入到LRU列表首部，而是放到LRU列表的midpoint位置。这个算法在InnoDB引擎下称为midpoint insertion strategy。在默认配置下，该位置在LRU列表长度的$\frac{5}{8}$处。midpoint位置由参数innodb_old_blocks_pct控制。在InnoDB中，把midpoint之后的位置称为old，之前的位置称为new。这么改进是因为有些操作会扫描所有页，但是他们也就只访问一次（例如索引操作或数据扫描操作）。另外一个参数innodb_old_blocks_time表示页读取到mid位置后过多久才会被加入到LRU列表的热端（前端）。   

InnoDB提供了压缩页的功能，即原本16KB的页压缩为1KB，2KB，4KB和8KB，采用unzip_LRU。对于压缩页的表，每个表的压缩比率可能各不相同，有的表页大小为8KB，有的表也大笑为2KB的情况。unzip_LRU申请4KB页大小，过程如下：  
1. 检查4KB的unzip_LRU列表，检查是否有可用的空闲页；
2. 若有则直接使用
3. 否则，检查8KB的unzip_LRU列表；
4. 若能够得到空闲页，将页分成两个4kb页，存放到4KB的unzip_LRU列表；
5. 若不能得到空闲页，从LRU列表申请一个16KB的页，将页分为1个8KB的页，2个4KB的页，分别放到对应的unzip_LRU列表中。


