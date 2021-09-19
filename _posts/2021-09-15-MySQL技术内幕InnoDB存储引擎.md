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

**重做日志缓冲**  
InnoDB存储引擎的内存区域除了有缓冲池外，还有重做日志缓冲。InnoDB引擎首先将重做日志信息放入到缓冲区，然后按照一定频率刷新到重做日志文件。这个区域一般不需要很大，8MB即可，该值由innodb_log_buffer_size控制。一般情况下，下面三个条件会刷新重做日志缓冲：  
* Master Thread 每一秒将重做日志缓冲刷新到重做日志文件
* 每个事务提交时会刷新
* 当buffer小于1/2时会刷新  

### Checkpoint 技术  
因为数据库内部使用缓冲来平衡磁盘和CPU速度，因此当服务器宕机后，必须有恢复数据的办法。当前事务数据库采用Write Ahead Log策略，即当事务提交时，先写重做日志，再修改页。  

检查点的作用是：当数据库发生宕机，数据库不需要重做所有日志，因为Checkpoint之前的页都已经刷新回磁盘里，故数据库只需对Checkpoint后的重做日志进行恢复。当LRU溢出最少的页时，若此页为脏页，那么需要强制执行Checkpoint，将脏页刷新到磁盘。  

对于InnoDB存储引擎而言，其是通过LSN（Log Sequence Number）来标记版本的。InnoDB有两种Checkpoint，分别为：Sharp Checkpoint和Fuzzy Checkpoint。Sharp Checkpoint发生在数据库关闭时将所有的脏页都刷新回磁盘，这是默认的工作方式；InnoDB内部使用Fuzzy Checkpoint，即只刷新一部分脏页。  

**Fuzzy Checkpoint发生时机**  
Master Thread差不多每秒或者每10秒从缓冲池的脏页列表中刷新一定比例的页回磁盘，这个过程是异步的。  
FLUSH_LRU_LIST Checkpoint 是因为InnoDB存储引擎要保证LRU中需要差不多100个空闲页可供使用。如果不足100个空闲页，则需要将脏页刷新回磁盘，此时会产生检查点。  
Async/Sync Flush Checkpoint，当重做日志不可用时，也就是重做日志过时了，需要将脏页写回，此时触发检查点（重做日志是循环使用的，检查点之前的日志可以被覆盖）  
Dirty Page too much ，脏页数量太多，检查点也会强制Checkpoint，其参数由innodb_max_dirty_pages_pct控制。  

### Master Thread 工作方式  
Master Thread具有最高的线程优先级别，其内部由多个循环组成：主循环、后台循环、刷新循环、暂停循环。  

**Loop（主循环）**  
大多数操作都在这个循环，其中有两大部分的操作：每秒钟的操作和每10秒钟的操作，伪代码如下：  
```c
void master_thread() {
loop:
    for(int i = 0;i<10;i++){
        // do thing once per second
        // sleep 1 second if necessary
    }
    do things once per ten seconds
    goto loop
}
```  
可以看到loop通过sleep实现，这意味着每秒一次或者每10秒1次的操作是不精确的，在负载很大的情况下会有延迟，只能说是大概频率。  

每秒的操作包括：  
* 日志缓冲刷新到磁盘，即使这个事务还没有提交
* 合并插入缓冲
* 至多刷新100个InnoDB的缓冲池中的脏页到磁盘
* 如果当前没有用户活动，则切换到background loop

即使某个事务没有提交，InnoDB存储引擎还是会每秒将重做日志缓冲中的内容刷新到重做日志文件，因此再大的事物cimmit的时间也是很短的。  
合并插入缓冲并不是每秒都会发生，InnoDB存储引擎会判断当前一秒内发生IO次数是否小于5，如果小于5次，InnoDB认为当前的IO压力小，可以执行合并插入缓冲操作(Insert Buffer)。  

每10秒的操作：  
* 刷新100个脏页到磁盘
* 合并至多5个插入缓冲
* 将日志缓冲刷新到磁盘
* 删除无用的undo页
* 刷新100个或者10个脏页到磁盘  

**background loop**
若当前没有用户活动或者数据库关闭(shutdown)，就会切换到这个循环。background loop会执行以下操作：  
* 删除无用的Undo页
* 合并20个插入缓冲
* 跳回主循环
* 不断刷新100个页面直到符合条件（可能，跳转到flush loop中完成）  

若flush loop也没什么事儿做，则会切换到suspend_loop，将Master Thread 挂起，等待事件发生。若用户启用了InnoDB引擎，却没有使用任何InnoDB引擎的表，那么Master Thread总是处于挂起的状态。  

### InnoDB 关键特性  
* 插入缓冲(Insert Buffer)  
* 两次写(Double Write)
* 自适应哈希索引(Adaptive Hash Index)
* 异步IO(Async IO)
* 刷新邻接页  

#### 插入缓冲  
InnoDB存储引擎中，主键是行唯一索引的标识符。通常应用程序中行记录的插入顺序是按照主键递增的顺序进行插入的。因此，插入**聚集索引**（Primary Key）一般是顺序的，不需要磁盘的随机读取，因此不需要随机读取另一个页，所以插入的速度是非常快的。  

然而更多时候一个表上有很多非聚集的辅助索引(secondary index)，比如用户按照name这个字段进行查找，而name字段建立了索引，且name字段不是唯一的，这样就产生了一个非聚集索引并且不是唯一索引（就是说同一个值的所有记录不放在一起，就是不聚集）。**在插入操作时，数据页的存放还是按主键a进行顺序存放的，但是对非聚集索引叶子节点的插入不再是顺序的了**（也就是比如插入1000个点，这些点的插入顺序可能是a,b,c,a,b,c,a,b,c，就需要不断循环访问a、b、c值对应的索引页），这就需要离散地访问非聚集索引页，由于随机读取的存在而导致了插入操作的性能下降。  

*解决方法*  
对于非聚集索引的插入或更新操作，不是每一次直接插入到索引页，而是先判断插入的非聚集索引页是否在缓冲池中，如果在则直接插入，否则先放入到一个Insert Buffer对象中，好似欺骗。数据库这个非聚集的索引已经插入到叶子节点，而实际上并没有，只是存放在另一个位置。然后再以一定的频率和情况进行Insert Buffer和辅助索引页子节点的merge(合并)操作，这样通常能将多个插入合并到一个操作中（因为在一个索引页中），这就大大提高了对于非聚集索引插入的性能。

*Insert Buffer 的使用满足的条件*  
* 索引是辅助索引(secondary index)
* 索引不是唯一的

*Change Buffer*  
InnoDB 后来引入Change Buffer，可将其视为Insert Buffer的扩展，也就是可以对DML操作的INSERT、DELETE、UPDATE都进行缓冲，它们分别是Insert Buffer，Delete Buffer，Purge Buffer。当然和之前的Insert Buffer一样，Change Buffer适用对象也依然是非唯一的辅助索引。  
对一条记录进行UPDATE操作分为两个过程：  
* 将记录标记为已删除
* 真正地将记录删除  
因此Delete Buffer对应UPDATE操作的第一个过程，即将记录标记为删除。Purge Buffer对应UPDATE操作的第二个过程，即将记录真正的删除。  

InnoDB存储引擎提供了参数innodb_change_buffering，用来开启各种Buffer的选项。该参数的可选的值为inserts, deletes, purges, changes, all, none。默认值为all。  

*内部实现*  
Insert Buffer是一个B+树，以前版本是每张表有一棵B+树，现在版本全局只有一棵B+树，负责对所有的辅助索引进行Insert Buffer。而这棵B+树存放在共享表空间中，默认也就是ibdata1中。因此，试图通过独立表空间ibd文件恢复表中数据时，往往会导致CHECK TABLE失败，这是因为表的辅助索引中的数据可能还在Insert Buffer中，也就是共享表空间中，所以通过idb文件进行恢复后，还需要进行REPAIR TABLE操作来重建表上的所有辅助索引。  

非叶节点：  
|space|marker|offset|
|:----:|:----:|:----:|  

叶子节点：  
|space|marker|offset|metadata|secondary index record|
|:----:|:----:|:----:|:----:|:----:|  

search key 一共占了9个字节，前四个字节是space表示插入记录所在表的表空间id，在InnoDB引擎中，每个表有唯一的space id，可以通过space id查询得知是哪张表。marker一个字节，用来兼容老版本的Inser Buffer。offset表示页所在的偏移量，4个字节（一个表由很多个物理页构成）。  
当辅助索引要插入到页(space, offset)时，如果这个页不在缓冲池，那么InnoDB存储引擎首先根据上述规则构造一个search key，接下来查询Insert Buffer这棵B+树，然后再将这条记录插入到叶子节点中。  

叶子节点前四个字段与之前相同，metadata占用4个字节，其中由如下构成：  
|名称|字节|
|:----:|:----:|
|IBUF_REC_OFFSET_COUNT|2|
|IBUF_REC_OFFSET_TYPE|1|
|IBUF_REC_OFFSET_FLAGS|1|

IBUF_REC_OFFSET_COUNT保存两个字节的证书，用来排序每个记录进入Insert Buffer的顺序，因为Change Buffer的需要，所以在进入insert buffer时按照顺序回放这个修改过程才能得到记录的正确值。  
第五列开始，就是实际插入记录的各个字段。为了保证每次Merge成功，还需要有一个特殊的页来标记每个辅助索引页(space, page_no)的可用空间，这个页的类型为Insert Buffer Bitmap。该Bitmao 用来追踪16384个辅助索引页，也就是256个区。  
![](/images/20210915-4.png)  

*Merge Insert Buffer*  
时机：  
* 辅助索引页被读取时，比如一个简单的SELECT操作，此时要检车BitMap页，然后确认该辅助索引页是否有记录存放于Insert Buffer B+树中，若有，则将B+树中的记录插入到该辅助索引页中。可以看到，对该页的多次操作通过一次操作合并到了原有的辅助索引页中，因此性能会有大幅提高。  
* Insert Buffer Bitmap 页追踪到该辅助索引页已经无可用空间时；
* Master Thread每秒和每10秒会执行一次，Master Thread中，执行merge 操作的不止一个页，而是根据srv_innodb_io_capacity的百分比来决定真正要合并多少个辅助索引页，随机选择。  

#### 两次写  
doublewrite 带给InnoDB存储引擎的是数据页的可靠性。  

> 当发生数据库宕机时，InnoDB正在写入某个页到表中，而这个页只写了一部分，比如16K的页写了前4K，之后发生宕机，这种情况被称为部分写失效。  

**原因**
然而通过重做日志进行恢复，这是一个办法，但是重做日志中记录的是对页的物理操作，如偏移量800，写'aaa'记录，如果这个页本身已经发生损坏，再对其进行重做没有意义（因为这个页上的所有数据都已经不可靠，很久以前写的数据可能会发生改变）。  

**组成**  
![](/images/20210915-5.png)  
1. 先将脏页写到内存的doublewrite buffer
2. doublewrite buffer分两次，每次1MB顺序写入共享表空间的物理磁盘上（两个分区按照顺序），立刻调用fsync，同步磁盘，在这个过程中doublewrite 页是连续的，因此这个过程开销不是很大。
3. 再将doublewrite buffer中的页写入到各个表空间文件，此时写入的页是离散的。  

若上面1和2发生宕机，物理页没有损坏，此时可以redo log进行重做，若3发生宕机，则可以用共享表空间的备份表进行恢复。    
一般主服务器master开启，slave关闭，或者有些文件系统本身提供了部分写失效，如ZFS文件系统，在这种情况下也可以不用启动doublewrite。

#### 自适应哈希索引  
InnoDB存储引擎会监控对各表上各索引页的查询。如果观潮到见咯哈希索引可以带来速度提升，则建立哈希索引，称之为自适应哈希索引(Adaptive Hash Index, AHI)。AHI是通过缓冲池的B+树页构造而来的，因此建立的速度很快，而且不需要对整张表结构建立哈希索引。InnoDB存储引擎会自动根据访问的频率和模式来自动地为某些热点页建立哈希索引。  

要求：  
访问模式必须都是WHERE a=xxx，也就是访问模式是一样的。AHI是存储引擎控制的，而不是用户控制的，用户只能控制打开或者禁用。

#### 异步IO  
用户发出一条IO操作，可以立即发送其他的IO操作，当全部IO操作发送完毕后，等待所有的IO操作完成，这就是AIO。AIO的优势时可以进行IO merge操作，也就是将多个IO合并为一个IO。例如  
用户需要访问页(space, page_no)：(8,6),(8,7),(8,8)，每个页16KB，那么同步 IO需要进行3次IO操作，而AIO会判断这三个页是连续的，因此AIO底层会发送一个IO请求，从(8,6)开始，读取48KB的页。  

#### 刷新邻接页  
当刷新一个脏页时，InnoDB会检测该页所在区(extent)的所有页，如果是脏页，那么一起进行刷新。这样做的好处显而易见，通过AIO可以将多个IO写入操作合并为一个IO操作，故在机械硬盘下有着显著的优势。  

但是传统固态硬盘有着较高的IOPS性能，则可以关闭该参数，innodb_flush_neighbors为0.  

















