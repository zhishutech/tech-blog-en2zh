# MyRocks Engine: Things to Know Before You Start
原文：[MySQL Query Performance: Not Just Indexes](https://www.percona.com/blog/2018/02/01/myrocks-engine-things-know-start/)

译者：魏新平

Percona 最近发布了Percona Server with MyRocks的GA版本。你能看到Facebook是怎样解释了在生产环境中使用MyRocks 取得的成功（直接翻译为你能够了解到为什么Facebook在生产环境使用MyRocks了是不是更好）。如果你使用[Percona repositories](https://www.percona.com/doc/percona-server/LATEST/myrocks/install.html) ，你能够简单的安装MyRocks插件并且用ps-admin --enable-rocksdb来启动它。

>Percona recently released Percona Server with MyRocks as GA. You can see how Facebook explains wins they see in production with MyRocks. Now if you use Percona repositories, you can simply install MyRocks plugin and enable it with ps-admin --enable-rocksdb. 

将它和典型的InnoDB进行比较时，存在一些主要和次要区别，我想在此强调一下。第一个主要的区别是MyRocks (based on RocksDB) 使用Log Structured Merge Tree数据结构，不是InnoDB的B+ tree数据结构。
>There are some major and minor differences when comparing it to typical InnoDB deployments, and I want to highlight them here. The first important difference is that MyRocks (based on RocksDB) uses Log Structured Merge Tree data structure, not a B+ tree like InnoDB.

你能够在我发布在DZone的[文章](https://dzone.com/articles/how-three-fundamental-data-structures-impact-stora) 当中了解到更多关于LSM引擎的信息 。总的来说，LSM引擎更适合写密集型的应用场景，读取速度可能会比较慢，全表扫描对于引擎来说负担会太重。当使用MyRocks作为应用底层时，需要特别注意这一点。MyRocks 不是加强版的InnoDB，也不能在所有应用场景下替换InnoDB。他有自己的优势/局限，就像InnoDB一样，你需要根据你数据的存取模式来选择使用哪一个引擎。

>You learn more about the LSM engine in my article for DZone.The summary is that an LSM data structure is good for write-intensive workloads, with the expense that reads might slow down (both point reads and especially range reads) and full table scans might be too heavy for the engine. This is important to keep in mind when designing applications for MyRocks. MyRocks is not an enhanced InnoDB, nor a one-size-fits-all replacement for InnoDB. It has its own pros/cons just like InnoDB. You need to decide which engine to use based on your applications data access patterns.

还有什么其他需要注意的区别的吗？
>What other differences should you be aware of?

让我们看一下目录结构。当前，所有的表和数据库都是存储在mysqldir的.rocksdb隐藏目录当中。名字和地址可以改变，但是所有的数据库当中的所有表还是存储在一系列的.sst文件当中，没有per-table / per-database的区分。

>Let’s look at the directory layout. Right now, all tables and all databases are stored in a hidden .rocksdb directory inside mysqldir. The name and location can be changed, but still all tables from all databases are stored in just a series of .sst files. There is no per-table / per-database separation.

默认情况下，MyRocks 使用LZ4来压缩所有的表。能够通过改变`rocksdb_default_cf_options`当中的变量来改变压缩的设置。默认值为`compression=kLZ4Compression;bottommost_compression=kLZ4Compression`。我们选择 LZ4，是因为它在很小的cpu负载下提供了可接受的压缩比。其他的压缩方式包括Zlib 和 ZSTD，或者直接不压缩。你能够在Peter和我的文章当中学习到更多关于[压缩比VS速度](https://www.percona.com/blog/2016/04/13/evaluating-database-compression-methods-update/)  的信息。为了比较装载了来自我自制路由器软件的流量统计数据的MyRocks表的物理大小，我使用了为pmacct收集器软件创建的下表。

>By default in Percona Server for MySQL, MyRocks will use LZ4 compression for all tables. You can change compression settings by changing the rocksdb_default_cf_options server variable. By default it set to compression=kLZ4Compression;bottommost_compression=kLZ4Compression. We chose LZ4 compression as it provides acceptable compression level with very little CPU overhead. Other possible compression methods are Zlib and ZSTD, or no compression at all. You can learn more about compression ratio vs. speed in Peter’s and my post.To compare the data size of a MyRocks table loaded with traffic statistic data from my homebrew router, I’ve used the following table created for pmacct collector:
```sql
CREATE TABLE `acct_v9` (
  `tag` int(4) unsigned NOT NULL,
  `class_id` char(16) NOT NULL,
  `class` varchar(255) DEFAULT NULL,
  `mac_src` char(17) NOT NULL,
  `mac_dst` char(17) NOT NULL,
  `vlan` int(2) unsigned NOT NULL,
  `as_src` int(4) unsigned NOT NULL,
  `as_dst` int(4) unsigned NOT NULL,
  `ip_src` char(15) NOT NULL,
  `ip_dst` char(15) NOT NULL,
  `port_src` int(2) unsigned NOT NULL,
  `port_dst` int(2) unsigned NOT NULL,
  `tcp_flags` int(4) unsigned NOT NULL,
  `ip_proto` char(6) NOT NULL,
  `tos` int(4) unsigned NOT NULL,
  `packets` int(10) unsigned NOT NULL,
  `bytes` bigint(20) unsigned NOT NULL,
  `flows` int(10) unsigned NOT NULL,
  `stamp_inserted` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `id` int(11) NOT NULL AUTO_INCREMENT,
  PRIMARY KEY (`id`)
) ENGINE=ROCKSDB AUTO_INCREMENT=20127562
```

正如你所看见的，表中有大概2000万条数据。MyRocks (用默认的 LZ4 压缩)使用了828MB。 InnoDB (默认，未压缩) 使用了3760MB。
>as you can see, there are about 20mln records in this table. MyRocks (with default LZ4 compression) uses 828MB. InnoDB (uncompressed) uses 3760MB.

你能够在 .rocksdb 目录的LOG文件当中找到RocksDB 实例的详细信息。查看这些日志，可以进行更详细的诊断。你也能够使用SHOW ENGINE ROCKSDB STATUS命令，但是这会比SHOW ENGINE INNODB STATUS返回的内容更复杂，需要消耗大量的精力和时间去理解。

>You can find very verbose information about your RocksDB instance in the LOG file located in .rocksdb directory. Check this file for more diagnostics. You can also try the SHOW ENGINE ROCKSDB STATUS command, but it is even more cryptic than SHOW ENGINE INNODB STATUS. It takes time to parse and to understand it.

注意，现在MyRocks只支持 READ-COMMITTED 隔离级别。并没有 REPEATABLE-READ 隔离级别，也没有像InnoDB里一样的gap锁。理论上，
RocksDB 只支持 SNAPSHOT 隔离级别。然而，MySQL 当中并没有SNAPSHOT 隔离级别的概念，所以我们没有实现特殊的语法去支持。如果你对这个感兴趣，请联系我们。
>Keep in mind that at this time MyRocks supports only READ-COMMITTED isolation levels. There is no REPEATABLE-READ isolation level and no gap locking like in InnoDB. In theory, RocksDB should support SNAPSHOT isolation level. However, there is no notion of SNAPSHOT isolation in MySQL so we have not implemented the special syntax to support this level. Please let us know if you would be interested in this.

当你试图加载大量的数据到MyRocks 当中时，你可能会遇到问题（不幸的是这个可能是你使用MyRocks 时的首次工作，当你使用LOAD DATA, INSERT INTO myrocks_table SELECT * FROM innodb_table 或者 ALTER TABLE innodb_table ENGINE=ROCKSDB）。假如你的表太大，并且你没有足够的内存，RocksDB 就会崩溃。在生产环境中，你应该为你加载数据的session设置rocksdb_bulk_load=1。了解更多请查看文章：https://github.com/facebook/mysql-5.6/wiki/data-loading。

>For bulk loads, you may face problems trying to load large amounts of data into MyRocks (and unfortunately this might be the very first operation when you start playing with MyRocks as you try to LOAD DATA, INSERT INTO myrocks_table SELECT * FROM innodb_table or ALTER TABLE innodb_table ENGINE=ROCKSDB). If your table is big enough and you do not have enough memory, RocksDB crashes. As a workaround, you should set rocksdb_bulk_load=1 for the session where you load data.  See more on this page: https://github.com/facebook/mysql-5.6/wiki/data-loading.

在MyRocks中的Block cache有点类似于innodb_buffer_pool_size，但是对于MyRocks它主要有利于读取数据。您可能需要调整rocksdb_block_cache_size设置。另外，它默认使用buffered reads，在这种情况下，操作系统的cache缓存着压缩的数据，而RockDB block cache 会缓存未压缩的数据。你可以保持这种两层的缓存机制，或者你可以修改rocksdb_use_direct_reads=ON关闭缓存，强制block cache直接读取。LSM树的本质要求当一层变满时，有一个合并过程将压缩数据推到下一层。这个过程可能相当密集并会影响用户查询速度。可以将其调整为不那么密集。

>Block cache in MyRocks is somewhat similar to innodb_buffer_pool_size, however for MyRocks it’s mainly beneficial for reads. You may want to tune the rocksdb_block_cache_size setting. Also keep in mind it uses buffered reads by default, and in this case the OS cache contains cached compressed data and RockDB block cache will contain uncompressed data. You may keep this setup to have two levels of cache, or you can disable buffering by forcing block cache to use direct reads with rocksdb_use_direct_reads=ON.
The nature of LSM trees requires that when a level becomes full, there is a merge process that pushes compacted data to the next level. This process can be quite intensive and affect user queries. It is possible to tune it to be less intensive.

现在还没有类似于Percona XtraBackup一样的热备软件来执行MyRocks表的热备份（我们正在研究这个）。你可以使用mysqldump 来进行逻辑备份，或者使用文件系统层面的snapshots ，比如LVM 或 ZFS 。

>Right now there is no hot backup software like Percona XtraBackup to perform a hot backup of MyRocks tables (we are looking into this). At this time you can use mysqldump for logical backups, or use filesystem-level snapshots like LVM or ZFS.

在我们的[官方文档](https://www.percona.com/doc/percona-server/LATEST/myrocks/limitations.html.)当中，你可以了解到更多关于MyRocks 的优势和局限性.

>You can find more MyRocks specifics and limitations in our docs at https://www.percona.com/doc/percona-server/LATEST/myrocks/limitations.html.

我们期待大家的反馈 。
>We are looking for feedback on your MyRocks experience!

##更新（2018-02-12）
在获得Facebook MyRocks 团队的反馈后，我对原来的文章进行了更新。
>UPDATES (12-Feb-2018)
Updates to the original post with the feedback provided by Facebook MyRocks team

1. 隔离级别
MyRocks 支持READ COMMITTED 和 REPEATABLE READ隔离级别，不支持 
SERIALIZABLE。想了解更详细的信息可以阅读https://github.com/facebook/mysql-5.6/wiki/Transaction-Isolation。MyRocks 实现REPETABLE READ的方法和InnoDB不一样 — MyRocks 使用类似PostgreSQL 的snapshot isolation。
在Percona server 中，不允许在MyRocks 表上使用 REPEATABLE READ 隔离级别，因为REPEATABLE READ 隔离级别在innodb和myrocks db上的处理方式不一样。
>1. Isolation Levels
MyRocks supports READ COMMITTED and REPEATABLE READ. MyRocks does not support SERIALIZABLE.
Please read https://github.com/facebook/mysql-5.6/wiki/Transaction-Isolation for details.
The way to implement REPETABLE READ was different from MyRocks and InnoDB — MyRocks used
PostgreSQL style snapshot isolation.
In Percona Server we do not allow REPEATABLE READ for MyRocks tables, as the behavior will be different from InnoDB.

2. 在线二进制备份工具
网上有一个开源的在线二进制备份工具—myrocks_hotabackup：https://github.com/facebook/mysql-5.6/blob/fb-mysql-5.6.35/scripts/myrocks_hotbackup
>2. Online Binary Backup Tool
There is an open source online binary backup tool for MyRocks — myrocks_hotabackup
https://github.com/facebook/mysql-5.6/blob/fb-mysql-5.6.35/scripts/myrocks_hotbackup
