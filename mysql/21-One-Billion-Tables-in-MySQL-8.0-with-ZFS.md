> 作者：Alexander Rubin
>
> 发布时间：2018-10-22
>
> 文章关键字：Big Data, MySQL, MySQL 8.0, ZFS，MySQL Scalability, Scalability
>
> 文章原文：[One Billion Tables in MySQL 8.0 with ZFS](https://www.percona.com/blog/2018/10/22/one-billion-tables-in-mysql-8-0-with-zfs/)

------------------

## The short version
## 摘要
I created > one billion InnoDB tables in MySQL 8.0 (tables, not rows) just for fun. Here is the proof:
我在 MySQL8.0上创建了10亿+张InnoDB表（注意是表而不是行），如下：

```
$ mysql -A
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 1425329
Server version: 8.0.12 MySQL Community Server - GPL

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> select count(*) from information_schema.tables;
+------------+
| count(*)   |
+------------+
| 1011570298 |
+------------+
1 row in set (6 hours 57 min 6.31 sec)
```

Yes, it took 6 hours and 57 minutes to count them all!
是的，它耗费了6小时57分钟去统计表数目！

## Why does anyone need one billion tables?
## 谁会需要创建10亿+张表？

In my previous blog post, I created and tested [MySQL 8.0 with 40 million tables](https://www.pgcon.org/2013/schedule/attachments/283_Billion_Tables_Project-PgCon2013.pdf)  (that was a real case study). The One Billion Tables project is not a real world scenario, however. I was challenged by [Billion Tables Project (BTP) in PostgreSQL](https://www.pgcon.org/2013/schedule/attachments/283_Billion_Tables_Project-PgCon2013.pdf), and decided to repeat it with MySQL, creating 1 billion InnoDB tables.

在我之前的文章中，我创建和测试了[MySQL 8.0上创建4000w张表](https://www.percona.com/blog/2018/09/03/40-million-tables-in-mysql-8-0-with-zfs/)（这是一个真实的案例）。不过10亿张表不是真实的案例场景，是因为我想挑战下[在PG上创建了10亿张表](https://www.pgcon.org/2013/schedule/attachments/283_Billion_Tables_Project-PgCon2013.pdf)的测试，所以准备在MySQL下创建下10亿张InnoDB表。

As an aside: I think MySQL 8.0 is the first MySQL version where creating 1 billion InnoDB tables is even practically possible.
注：我认为MySQL8.0才是第一个具有创建10亿张InnoDB表可能性的MySQL版本。

## Challenges with one billion InnoDB tables
## 挑战10亿张InnoDB表

### Disk space
### 磁盘空间

The first and one of the most important challenges is disk space. InnoDB allocates data pages on disk when creating .ibd files. Without disk level compression we need > 25Tb of disk. The good news: we have ZFS which provides transparent disk compression. Here’s how the disk utilization looks:

首先面临第一个也是最重要的挑战就是磁盘空间。创建.ibd文件时，InnoDB在磁盘上分配数据页。如果不做磁盘压缩，我们至少需要25T的存储容量。不过好消息是：我们的ZFS提供透明的磁盘压缩。以下是磁盘利用率的表现：

Actual data (apparent-size):
实际大小：

```
# du -sh --apparent-size /mysqldata/
26T     /mysqldata/
```

Compressed data:
压缩后：

```
# du -sh /mysqldata/
2.4T    /mysqldata/
```

Compression ratio:
压缩率：

```
# zfs get compression,compressratio
...
mysqldata/mysql/data             compressratio         7.14x                      -
mysqldata/mysql/data             compression           gzip                       inherited from mysqldata/mysql
```

(Looks like the compression ratio reported is not 100% correct, we expect ~10x compression ratio.)
看起来报告不是100%准确，我们达到了10倍+的压缩率

### Too many tiny files
### 许多小文件

This is usually the big issue with databases that create a file per table. With MySQL 8.0 we can create a shared tablespace and “assign” a table to it. I created a tablespace per database, and created 1000 tables in each database.

为每张表要创建一个表空间文件，这是大问题。不过在MySQL 8.0中，我们可以创建一个通用表空间（General Tablespace）并在创建表时将表”分配“到表空间上。这里我为每个database创建一个通用表空间，每个database上创建了1000张表。

The result:
结果就是：

```
mysql> select count(*) from information_schema.schema;
+----------+
| count(*) |
+----------+
|  1011575 |
+----------+
1 row in set (1.31 sec)
```

### Creating tables
### 创建表
Another big challenge is how to create tables fast enough so it will not take months. I have used three approaches:
1. Disabled all possible consistency checks in MySQL, and decreased the innodb page size to 4K (these config options are NOT for production use)
2. Created tables in parallel: as the [mutex contention bug in MySQL 8.0](https://bugs.mysql.com/bug.php?id=87827) has been fixed, creating tables in parallel works fine.
3. Use local NVMe cards on top of an AWS ec2 i3.8xlarge instance

另一个挑战点就是如何快速的创建表从而避免我们要耗费数月的时间。我用了三个锦囊妙计：
1. 禁用MySQL里面一切可能的一致性检测，减小innodb的page大小为4k（这些配置更改不适合生产环境）
2. 并发创建表。因为之前[MySQL 8.0中的互斥量争用问题](https://bugs.mysql.com/bug.php?id=87827)已经得到修复，所以并发创建表表现良好。
3. 在AWS ec2 i3.8 xlarge的实例上使用本地的NVMe卡

my.cnf config file (I repeat: do not use this in production):
my.cnf的配置信息如下（重申一遍：不要直接用在生产上）：

```
[mysqld]
default-authentication-plugin = mysql_native_password
performance_schema=0
datadir=/mysqldata/mysql/data
socket=/mysqldata/mysql/data/mysql.sock
log-error = /mysqldata/mysql/log/error.log
skip-log-bin=1
innodb_log_group_home_dir = /mysqldata/mysql/log/
innodb_doublewrite = 0
innodb_checksum_algorithm=none
innodb_log_checksums=0
innodb_flush_log_at_trx_commit=0
innodb_log_file_size=2G
innodb_buffer_pool_size=100G
innodb_page_size=4k
innodb_flush_method=nosync
innodb_io_capacity_max=20000
innodb_io_capacity=5000
innodb_buffer_pool_instances=32
innodb_stats_persistent = 0
tablespace_definition_cache = 524288
schema_definition_cache = 524288
table_definition_cache = 524288
table_open_cache=524288
table_open_cache_instances=32
open-files-limit=1000000
```

ZFS pool:

```
# zpool status
  pool: mysqldata
 state: ONLINE
  scan: scrub repaired 0B in 1h49m with 0 errors on Sun Oct 14 02:13:17 2018
config:
        NAME        STATE     READ WRITE CKSUM
        mysqldata   ONLINE       0     0     0
          nvme0n1   ONLINE       0     0     0
          nvme1n1   ONLINE       0     0     0
          nvme2n1   ONLINE       0     0     0
          nvme3n1   ONLINE       0     0     0
errors: No known data errors
```

A simple “deploy” script to create tables in parallel (includes the sysbench table structure):
一个简单的并发创建表的脚本（表结构使用了sysbench里面的表）：
```
#/bin/bash
function do_db {
        db_exist=$(mysql -A -s -Nbe "select 1 from information_schema.schemata where schema_name = '$db'")
        if [ "$db_exist" == "1" ]; then echo "Already exists: $db"; return 0; fi;
        tbspace="create database $db; use $db; CREATE TABLESPACE $db ADD DATAFILE '$db.ibd' engine=InnoDB";
        #echo "Tablespace $db.ibd created!"
        tables=""
        for i in {1..1000}
        do
                table="CREATE TABLE sbtest$i ( id int(10) unsigned NOT NULL AUTO_INCREMENT, k int(10) unsigned NOT NULL DEFAULT '0', c varchar(120) NOT NULL DEFAULT '', pad varchar(60) NOT NULL DEFAULT '', PRIMARY KEY (id), KEY k_1 (k) ) ENGINE=InnoDB DEFAULT CHARSET=latin1 tablespace $db;"
                tables="$tables; $table;"
        done
        echo "$tbspace;$tables" | mysql
}
c=0
echo "starting..."
c=$(mysql -A -s -Nbe "select max(cast(SUBSTRING_INDEX(schema_name, '_', -1) as unsigned)) from information_schema.schemata where schema_name like 'sbtest_%'")
for m in {1..100000}
do
        echo "m=$m"
        for i in {1..30}
        do
                let c=$c+1
                echo $c
                db="sbtest_$c"
                do_db &
        done
        wait
done
```

How fast did we create tables? Here are some stats:
我们创建表有多快呢？可以通过下面的状态量观测：
```
# mysqladmin -i 10 -r ex|grep Com_create_table
...
| Com_create_table                                      | 6497                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Com_create_table                                      | 6449
```

So we created ~650 tables per second. The average, above, is per 10 seconds.
我们约每秒创建650张表，上面是每10秒创建的表数量。

## Counting the tables
## 统计表数量
It took > 6 hours to do “count(*) from information_schema.tables”! Here is why:

之前我们通过"count(*) from information_schema.tables"方式查看表数量耗费了6个多小时。因为：

1. MySQL 8.0 uses a new data dictionary (this is great as it avoids creating 1 billion frm files). Everything is stored in this file:
----
1. MySQL 8.0 使用了一个新的数据字典（这很妙，避免创建10亿个frm文件）。所有的内容都存储在下面这个文件里：

```
# ls -lah /mysqldata/mysql/data/mysql.ibd
-rw-r----- 1 mysql mysql 6.1T Oct 18 15:02 /mysqldata/mysql/data/mysql.ibd
```

2. The information_schema.tables is actually a view:
----
2. information_schema.tables实际上是一个视图：
```
mysql> show create table information_schema.tables\G
*************************** 1. row ***************************
                View: TABLES
         Create View: CREATE ALGORITHM=UNDEFINED DEFINER=`mysql.infoschema`@`localhost` SQL SECURITY DEFINER VIEW `information_schema`.`TABLES` AS select `cat`.`name` AS `TABLE_CATALOG`,`sch`.`name` AS `TABLE_SCHEMA`,`tbl`.`name` AS `TABLE_NAME`,`tbl`.`type` AS `TABLE_TYPE`,if((`tbl`.`type` = 'BASE TABLE'),`tbl`.`engine`,NULL) AS `ENGINE`,if((`tbl`.`type` = 'VIEW'),NULL,10) AS `VERSION`,`tbl`.`row_format` AS `ROW_FORMAT`,internal_table_rows(`sch`.`name`,`tbl`.`name`,if(isnull(`tbl`.`partition_type`),`tbl`.`engine`,''),`tbl`.`se_private_id`,(`tbl`.`hidden` <> 'Visible'),`ts`.`se_private_data`,coalesce(`stat`.`table_rows`,0),coalesce(cast(`stat`.`cached_time` as unsigned),0)) AS `TABLE_ROWS`,internal_avg_row_length(`sch`.`name`,`tbl`.`name`,if(isnull(`tbl`.`partition_type`),`tbl`.`engine`,''),`tbl`.`se_private_id`,(`tbl`.`hidden` <> 'Visible'),`ts`.`se_private_data`,coalesce(`stat`.`avg_row_length`,0),coalesce(cast(`stat`.`cached_time` as unsigned),0)) AS `AVG_ROW_LENGTH`,internal_data_length(`sch`.`name`,`tbl`.`name`,if(isnull(`tbl`.`partition_type`),`tbl`.`engine`,''),`tbl`.`se_private_id`,(`tbl`.`hidden` <> 'Visible'),`ts`.`se_private_data`,coalesce(`stat`.`data_length`,0),coalesce(cast(`stat`.`cached_time` as unsigned),0)) AS `DATA_LENGTH`,internal_max_data_length(`sch`.`name`,`tbl`.`name`,if(isnull(`tbl`.`partition_type`),`tbl`.`engine`,''),`tbl`.`se_private_id`,(`tbl`.`hidden` <> 'Visible'),`ts`.`se_private_data`,coalesce(`stat`.`max_data_length`,0),coalesce(cast(`stat`.`cached_time` as unsigned),0)) AS `MAX_DATA_LENGTH`,internal_index_length(`sch`.`name`,`tbl`.`name`,if(isnull(`tbl`.`partition_type`),`tbl`.`engine`,''),`tbl`.`se_private_id`,(`tbl`.`hidden` <> 'Visible'),`ts`.`se_private_data`,coalesce(`stat`.`index_length`,0),coalesce(cast(`stat`.`cached_time` as unsigned),0)) AS `INDEX_LENGTH`,internal_data_free(`sch`.`name`,`tbl`.`name`,if(isnull(`tbl`.`partition_type`),`tbl`.`engine`,''),`tbl`.`se_private_id`,(`tbl`.`hidden` <> 'Visible'),`ts`.`se_private_data`,coalesce(`stat`.`data_free`,0),coalesce(cast(`stat`.`cached_time` as unsigned),0)) AS `DATA_FREE`,internal_auto_increment(`sch`.`name`,`tbl`.`name`,if(isnull(`tbl`.`partition_type`),`tbl`.`engine`,''),`tbl`.`se_private_id`,(`tbl`.`hidden` <> 'Visible'),`ts`.`se_private_data`,coalesce(`stat`.`auto_increment`,0),coalesce(cast(`stat`.`cached_time` as unsigned),0),`tbl`.`se_private_data`) AS `AUTO_INCREMENT`,`tbl`.`created` AS `CREATE_TIME`,internal_update_time(`sch`.`name`,`tbl`.`name`,if(isnull(`tbl`.`partition_type`),`tbl`.`engine`,''),`tbl`.`se_private_id`,(`tbl`.`hidden` <> 'Visible'),`ts`.`se_private_data`,coalesce(cast(`stat`.`update_time` as unsigned),0),coalesce(cast(`stat`.`cached_time` as unsigned),0)) AS `UPDATE_TIME`,internal_check_time(`sch`.`name`,`tbl`.`name`,if(isnull(`tbl`.`partition_type`),`tbl`.`engine`,''),`tbl`.`se_private_id`,(`tbl`.`hidden` <> 'Visible'),`ts`.`se_private_data`,coalesce(cast(`stat`.`check_time` as unsigned),0),coalesce(cast(`stat`.`cached_time` as unsigned),0)) AS `CHECK_TIME`,`col`.`name` AS `TABLE_COLLATION`,internal_checksum(`sch`.`name`,`tbl`.`name`,if(isnull(`tbl`.`partition_type`),`tbl`.`engine`,''),`tbl`.`se_private_id`,(`tbl`.`hidden` <> 'Visible'),`ts`.`se_private_data`,coalesce(`stat`.`checksum`,0),coalesce(cast(`stat`.`cached_time` as unsigned),0)) AS `CHECKSUM`,if((`tbl`.`type` = 'VIEW'),NULL,get_dd_create_options(`tbl`.`options`,if((ifnull(`tbl`.`partition_expression`,'NOT_PART_TBL') = 'NOT_PART_TBL'),0,1))) AS `CREATE_OPTIONS`,internal_get_comment_or_error(`sch`.`name`,`tbl`.`name`,`tbl`.`type`,`tbl`.`options`,`tbl`.`comment`) AS `TABLE_COMMENT` from (((((`mysql`.`tables` `tbl` join `mysql`.`schemata` `sch` on((`tbl`.`schema_id` = `sch`.`id`))) join `mysql`.`catalogs` `cat` on((`cat`.`id` = `sch`.`catalog_id`))) left join `mysql`.`collations` `col` on((`tbl`.`collation_id` = `col`.`id`))) left join `mysql`.`tablespaces` `ts` on((`tbl`.`tablespace_id` = `ts`.`id`))) left join `mysql`.`table_stats` `stat` on(((`tbl`.`name` = `stat`.`table_name`) and (`sch`.`name` = `stat`.`schema_name`)))) where (can_access_table(`sch`.`name`,`tbl`.`name`) and is_visible_dd_object(`tbl`.`hidden`))
character_set_client: utf8
collation_connection: utf8_general_ci
```

and the explain plan looks like this:
而且通过explain看到它的执行计划如下：

```
mysql> explain select count(*) from information_schema.tables \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: cat
   partitions: NULL
         type: index
possible_keys: PRIMARY
          key: name
      key_len: 194
          ref: NULL
         rows: 1
     filtered: 100.00
        Extra: Using index
*************************** 2. row ***************************
           id: 1
  select_type: SIMPLE
        table: tbl
   partitions: NULL
         type: ALL
possible_keys: schema_id
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 1023387060
     filtered: 100.00
        Extra: Using where; Using join buffer (Block Nested Loop)
*************************** 3. row ***************************
           id: 1
  select_type: SIMPLE
        table: sch
   partitions: NULL
         type: eq_ref
possible_keys: PRIMARY,catalog_id
          key: PRIMARY
      key_len: 8
          ref: mysql.tbl.schema_id
         rows: 1
     filtered: 11.11
        Extra: Using where
*************************** 4. row ***************************
           id: 1
  select_type: SIMPLE
        table: stat
   partitions: NULL
         type: eq_ref
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 388
          ref: mysql.sch.name,mysql.tbl.name
         rows: 1
     filtered: 100.00
        Extra: Using index
*************************** 5. row ***************************
           id: 1
  select_type: SIMPLE
        table: ts
   partitions: NULL
         type: eq_ref
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 8
          ref: mysql.tbl.tablespace_id
         rows: 1
     filtered: 100.00
        Extra: Using index
*************************** 6. row ***************************
           id: 1
  select_type: SIMPLE
        table: col
   partitions: NULL
         type: eq_ref
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 8
          ref: mysql.tbl.collation_id
         rows: 1
     filtered: 100.00
        Extra: Using index
```

## Conclusions
## 结论

1. I have created more than 1 billion real InnoDB tables with indexes in MySQL 8.0, just for fun, and it worked. It took ~2 weeks to create.
2. Probably MySQL 8.0 is the first version where it is even practically possible to create billion InnoDB tables
3. ZFS compression together with NVMe cards makes it reasonably cheap to do, for example, by using i3.4xlarge or i3.8xlarge instances on AWS.

-------
1. 只是因为个人兴趣，我在MySQL 8.0上创建了10亿张InnoDB表和索引，我成功了。它花费了我大约2周的时间。
2. 大概率MySQL 8.0是MySQL里面第一个支持能够创建10亿张InnoDB表的版本。
3. ZFS 的压缩再结合NVMe卡，可以降低成本。例如，选择AWS的i3.4xlarge或者i3.8xlarge实例。

![MySQL 10亿！](https://www.percona.com/blog/wp-content/uploads/2018/10/one-billion-tables-mysql-367x250.jpg)


