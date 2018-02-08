# MySQL 查询性能优化：不只是索引


在本文当中，我将研究优化索引是否总是提高MySQL查询性能的关键。（剧透一下，不是）

当我们优化MySQL查询语句的时候，我们首先关心的通常是该查询是否使用了正确的索引来获取数据。如果获取数据永远是语句执行当中最耗时的操作，那么这种思路没有问题。但是，情况并非总是如此。

让我们看一下下面这个例子：
```sql
mysql> show create table tbl G
*************************** 1. row ***************************
      Table: tbl
Create Table: CREATE TABLE `tbl` (
 `id` int(11) NOT NULL AUTO_INCREMENT,
 `k` int(11) NOT NULL DEFAULT '0',
 `g` int(10) unsigned NOT NULL,
 PRIMARY KEY (`id`),
 KEY `k_1` (`k`)
) ENGINE=InnoDB AUTO_INCREMENT=2340933 DEFAULT CHARSET=latin1
1 row in set (0.00 sec)
mysql> explain select g,count(*) c from tbl where k<1000000 group by g having c>7 G
*************************** 1. row ***************************
          id: 1
 select_type: SIMPLE
       table: tbl
  partitions: NULL
        type: ALL
possible_keys: k_1
         key: NULL
     key_len: NULL
         ref: NULL
        rows: 998490
    filtered: 50.00
       Extra: Using where; Using temporary; Using filesort
1 row in set, 1 warning (0.00 sec)
mysql> select g,count(*) c from tbl where k<1000000 group by g having c>7;
+--------+----+
| g      | c  |
+--------+----+
|  28846 |  8 |
| 139660 |  8 |
| 153286 |  8 |
...
| 934984 |  8 |
+--------+----+
22 rows in set (6.80 sec)
```
当看到这样的语句，很多人会认为主要的原因是全表扫描。那么有人就会奇怪了，为什么MySQL的优化器不使用索引呢（那是因为该语句查询的数据不具备选择性，简单点说就是获取的数据相对于整张表来说太多了）这样的想法就会导致有些人强制使用索引，但是效果只会更差。
```sql
mysql> select g,count(*) c from tbl force index(k) where k<1000000 group by g having c>7;
+--------+----+
| g      | c  |
+--------+----+
|  28846 |  8 |
| 139660 |  8 |
...
| 934984 |  8 |
+--------+----+
22 rows in set (9.37 sec)
```
或许有些人会把基于k列的单列索引扩展成基于k和g列的联合索引。事实上，这样并不会有任何效果。
```sql
mysql> alter table tbl drop key k_1, add key(k,g);
Query OK, 0 rows affected (5.35 sec)
Records: 0  Duplicates: 0  Warnings: 0
 
mysql> explain select g,count(*) c from tbl where k<1000000 group by g having c>7 G
*************************** 1. row ***************************
          id: 1
 select_type: SIMPLE
       table: tbl
  partitions: NULL
        type: range
possible_keys: k
         key: k
     key_len: 4
         ref: NULL
        rows: 499245
    filtered: 100.00
       Extra: Using where; Using index; Using temporary; Using filesort
1 row in set, 1 warning (0.00 sec)
 
mysql> select g,count(*) c from tbl where k<1000000 group by g having c>7;
+--------+----+
| g      | c  |
+--------+----+
|  28846 |  8 |
| 139660 |  8 |
...
| 915436 |  8 |
| 934984 |  8 |
+--------+----+
22 rows in set (6.80 sec)
```
上述两种错误的优化思路完全是因为我们一直在错误的方向浪费精力：即迅速获取满足k<1000000的行。然而真正的问题并不是快速获取这些数据。如果我们把group by子句去掉，我们会的发现执行速度快了惊人的10倍。
```sql
mysql> select sum(g) from tbl where k<1000000;
+--------------+
| sum(g)       |
+--------------+
| 500383719481 |
+--------------+
1 row in set (0.68 sec)
```
针对这个特殊的语句，是否使用索引获取数据并不是主要的问题，我们应该关注于如何优化GROUP BY，这才是问题的所在。
在我的下篇博客当中，我将介绍4种MySQL执行GROUP BY的方法，来帮助更好的优化类似语句。

