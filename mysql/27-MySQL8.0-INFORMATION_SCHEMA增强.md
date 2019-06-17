# MySQL 8.0: Improvements to Information_schema

> 作者：[Gopal Shankar](https://mysqlserverteam.com/author/gopal/)
>
> 原文地址：<https://mysqlserverteam.com/mysql-8-0-improvements-to-information_schema/>
>
> 翻译：徐晨亮

*Coinciding with the new native data dictionary in MySQL 8.0, we have made a number of useful enhancements to our `INFORMATION_SCHEMA` subsystem design in MySQL 8.0. In this post I will first go through our legacy implementation as it has stood since MySQL 5.1, and then cover what’s changed.*



与MySQL 8.0原生数据字典一致，在MySQL 8.0的`INFORMATION_SCHEMA`子系统设计中，我们做了一些很有用的增强。在这篇文章中，我将会介绍自MySQL 5.1以来的旧的实现方式，然后介绍我们做了什么改变。

### Background

*`INFORMATION_SCHEMA` was first introduced into MySQL 5.0, as a standards compliant way of retrieving meta data from a running MySQL server. When we look at the history of `INFORMATION_SCHEMA` there have been a number of complaints about the performance of certain queries, particularly in the case that there are many database objects (schemas, tables etc).*



`INFORMATION_SCHEMA`首次引入MySQL 5.0，作为一种从运行的MySQL服务器检索元数据的标准兼容方式。当我们查看`INFORMATION_SCHEMA`的历史时，有一些关于某些查询性能的抱怨，特别是在有许多数据库对象（schema，表等）的情况下

*In an effort to address these reported issues, since MySQL 5.1 we have made a number of performance optimizations to speed up the execution of `INFORMATION_SCHEMA` queries. The optimizations are [described in the MySQL manual](http://dev.mysql.com/doc/refman/5.7/en/information-schema-optimization.html), and apply when the user provides an explicit schema name or table name in the query.*



为了解决这些上报的问题，从MySQL 5.1开始，我们进行了许多性能优化来加快`INFORMATION_SCHEMA`查询的执行速度。[MySQL手册](http://dev.mysql.com/doc/refman/5.7/en/information-schema-optimization.html)中描述了这些优化，并在用户在查询中提供显式schema名称或表名时应用。

*Alas, despite these improvements `INFORMATION_SCHEMA` performance is still a major pain point for many of our users. The key reason behind these performance issues in the current `INFORMATION_SCHEMA` implementation is that `INFORMATION_SCHEMA` tables are implemented as temporary tables that are created on-the-fly during query execution. These temporary tables are populated via:*

1. Meta data from files, e.g. table definitions from .FRM files.
2. Details from storage engines, e.g. dynamic table statistics.
3. Data from global data structures in the MySQL server.



尽管有这些改进，`INFORMATION_SCHEMA的`性能仍然是我们许多用户的主要痛点。当前`INFORMATION_SCHEMA`实现中这些性能问题背后的关键原因是`INFORMATION_SCHEMA`表的查询实现方式是，在查询执行期间即时创建的临时表。这些临时表通过以下方式填充：

1. 元数据来自文件，例如：表定义来自FRM文件
2. 细节来自于存储引擎，例如：动态表的统计信息
3. 来自MySQL server层中全局数据结构的数据

*For a MySQL server having hundreds of database, each with hundreds of tables within them, the `INFORMATION_SCHEMA` query would end-up doing lot of I/O reading each individual FRM files from the file system. And it would also end-up using more CPU cycles in effort to open the table and prepare related in-memory data structures. It does attempt to use the MySQL server table cache (the system variable ‘`table_definition_cache`‘), however in large server instances it’s very rare to have a table cache that is large enough to accommodate all of these tables.*



对于一个MySQL实例来说可能有上百个库，每个库又有上百张表，`INFORMATION_SCHEMA`查询最终会从文件系统中读取每个单独的FRM文件，造成很多I/O读取。并且最终还会使用更多的CPU周期来打开表并准备相关的内存数据结构。它确实尝试使用MySQL server层的表缓存（系统变量`table_definition_cache` ），但是在大型实例中，很少有一个足够大的表缓存来容纳所有的表。

*One can easily face the above mentioned performance issue if the optimization is not used by the `INFORMATION_SCHEMA` query. For example, let us consider the two queries below*



如果`INFORMATION_SCHEMA`查询未使用优化，则可以很容易碰到上面的性能问题。例如，让我们考虑下面的两个查询，

```mysql
mysql> EXPLAIN SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES WHERE
    -> TABLE_SCHEMA = 'test' AND TABLE_NAME = 't1'\G
    *************************** 1. row ***************************
           id: 1
      select_type: SIMPLE
        table: TABLES
       partitions: NULL
         type: ALL
    possible_keys: NULL
          key: TABLE_SCHEMA,TABLE_NAME
      key_len: NULL
          ref: NULL
         rows: NULL
     filtered: NULL
        Extra: Using where; Skip_open_table; Scanned 0 databases
    1 row in set, 1 warning (0.00 sec)

    mysql> EXPLAIN SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES WHERE
    -> TABLE_SCHEMA like 'test%' AND TABLE_NAME like 't%'\G
    *************************** 1. row ***************************
           id: 1
      select_type: SIMPLE
        table: TABLES
       partitions: NULL
         type: ALL
    possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: NULL
     filtered: NULL
        Extra: Using where; Skip_open_table; Scanned all databases
    1 row in set, 1 warning (0.00 sec)
```

*As we can see from the `EXPLAIN` output, we see that the former query would use the values provided in `WHERE` clause for the `TABLE_SCHEMA` and `TABLE_NAME` field as a key to read the desired `FRM` files from the file system. However, the latter query would end up reading all the `FRM` in the entire data directory, which is very costly and does not scale.*



从`EXPLAIN`的输出可以看到。我们看到前一个查询将使用`WHERE`子句中为`TABLE_SCHEMA`和`TABLE_NAME`字段提供的值作为从文件系统读取所需`FRM`文件的键。但是，后一个查询最终将读取整个数据目录中的所有`FRM`，这非常昂贵且无法扩展。

### Changes in MySQL 8.0

*One of the major changes in 8.0 is the introduction of a native data dictionary based on InnoDB.  This change has enabled us to get rid of file-based metadata store (`FRM` files) and also help MySQL to move towards supporting transactional DDL. For more details on introduction of data dictionary feature in 8.0 and its benefits, please look at Staale’s [post here](http://mysqlserverteam.com/a-new-data-dictionary-for-mysql/).*



8.0中的一个主要变化是引入了基于InnoDB的数据字典。这一变化使我们能够摆脱基于文件的元数据存储（`FRM`文件），并帮助MySQL转向支持事务DDL。有关在8.0中引入数据字典功能及其优点的更多详细信息，请在[此处](http://mysqlserverteam.com/a-new-data-dictionary-for-mysql/)查看Staale的文章。

*Now that the metadata of all database tables is stored in transactional data dictionary tables, it enables us to design an `INFORMATION_SCHEMA` table as a database VIEW over the data dictionary tables. This eliminates costs such as the creation of temporary tables for each `INFORMATION_SCHEMA` query during execution on-the-fly, and also scanning file-system directories to find FRM files. It is also now possible to utilize the full power of the MySQL optimizer to prepare better query execution plans using indexes on data dictionary tables.*



既然所有数据库表的元数据都存储在事务数据字典表中，它使我们能够将`INFORMATION_SCHEMA`表设计为数据字典表中的数据库VIEW。这消除了成本，例如在即时执行期间为每个`INFORMATION_SCHEMA`查询创建临时表，以及扫描文件系统目录以查找FRM文件。现在还可以利用MySQL优化器的全部功能，使用数据字典表上的索引来获得更好的执行计划。

*The following diagram explains the difference in design in MySQL 5.7 and 8.0.*



下面的图解释了MySQL 5.8和8.0设计上的区别

[![overview_of_is](http://mysqlserverteam.com/wp-content/uploads/2016/09/overview_of_IS-1024x724.jpg)](http://mysqlserverteam.com/wp-content/uploads/2016/09/overview_of_IS.jpg)If we consider the above example under *Background*, we see that the optimizer plans to use indexes on data dictionary tables, in both the cases.



如果我们之前介绍的背景下考虑上面的例子，我们会看到优化器计划在两种情况下都使用数据字典表上的索引。

```mysql
mysql> EXPLAIN SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA = 'test' AND TABLE_NAME = 't1';
  +--+-----------+-----++------+------------------+----------++----------------------+----+--------+----------------------------------+
  |id|select_type|table||type  |possible_keys     |key       ||ref                   |rows|filtered|Extra                             |
  +--+-----------+-----++------+------------------+----------++----------------------+----+--------+----------------------------------+
  | 1|SIMPLE     |cat  ||index |PRIMARY           |name      ||NULL                  |   1|  100.00|Using index                       |
  | 1|SIMPLE     |sch  ||eq_ref|PRIMARY,catalog_id|catalog_id||mysql.cat.id,const    |   1|  100.00|Using index                       |
  | 1|SIMPLE     |tbl  ||eq_ref|schema_id         |schema_id ||mysql.sch.id,const    |   1|   10.00|Using index condition; Using where|
  | 1|SIMPLE     |col  ||eq_ref|PRIMARY           |PRIMARY   ||mysql.tbl.collation_id|   1|  100.00|Using index                       |
  +--+-----------+-----++------+------------------+----------++----------------------+----+--------+----------------------------------+

  mysql> EXPLAIN SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA like 'test%' AND TABLE_NAME like 't%';
  +--+-----------+-----++------+------------------+----------++-----------------------+----+--------+---------------------------------+
  |id|select_type|table||type  |possible_keys     |key       || ref                   |rows|filtered|Extra                            |
  +--+-----------+-----++------+------------------+----------++-----------------------+----+--------+---------------------------------+
  | 1|SIMPLE     |cat  ||index |PRIMARY           |name      || NULL                  |   1|  100.00|Using index                      |
  | 1|SIMPLE     |sch  ||ref   |PRIMARY,catalog_id|catalog_id|| mysql.cat.id          |   6|   16.67|Using where; Using index         |
  | 1|SIMPLE     |tbl  ||ref   |schema_id         |schema_id || mysql.sch.id          |  26|    1.11|Using index condition;Using where|
  | 1|SIMPLE     |col  ||eq_ref|PRIMARY           |PRIMARY   || mysql.tbl.collation_id|   1|  100.00|Using index                      |
  +--+-----------+-----++------+------------------+----------++-----------------------+----+--------+---------------------------------+
```

*When we look at performance gain with this new `INFORMATION_SCHEMA` design in 8.0, we see that it is much more efficient than MySQL 5.7. As an example, **this query is now ~100 times faster** (with 100 databases with 50 tables each). A separate blog will describe more about performance of `INFORMATION_SCHEMA` in 8.0.*



当我们在8.0中使用这个新的`INFORMATION_SCHEMA`设计来查看性能提升时，我们发现它比MySQL 5.7更有效。例如，**此查询现在快〜100倍**（100个数据库，每个50个表）。另外一篇博客将详细介绍8.0 中`INFORMATION_SCHEMA`性能  。

```mysql
SELECT TABLE_SCHEMA, TABLE_NAME, TABLE_TYPE, ENGINE, ROW_FORMAT
  FROM information_schema.tables
  WHERE TABLE_SCHEMA LIKE 'db%';
```



#### Sources of Metadata

*Not all the `INFORMATION_SCHEMA` tables are implemented as a VIEW over the data dictionary tables in 8.0. Currently we have the following `INFORMATION_SCHEMA` tables designed as views:*



并非所有`INFORMATION_SCHEMA`表都通过8.0中的数据字典表作为VIEW实现。目前，我们将以下`INFORMATION_SCHEMA`表设计为视图：

- SCHEMATA
- TABLES
- COLUMNS
- VIEWS
- CHARACTER_SETS
- COLLATIONS
- COLLATION_CHARACTER_SET_APPLICABILITY
- STATISTICS
- KEY_COLUMN_USAGE
- TABLE_CONSTRAINTS



*Upcoming MySQL 8.0 versions aims to provide even the following `INFORMATION_SCHEMA` tables as views:*



即将推出的MySQL 8.0版本将提供以下  `INFORMATION_SCHEMA`表作为视图：

- EVENTS
- TRIGGERS
- ROUTINES
- REFERENTIAL_CONSTRAINTS

*To describe the `INFORMATION_SCHEMA` queries which are not directly implemented as VIEWs over data dictionary tables, let me first describe that there are two types of meta data which are presented in `INFORMATION_SCHEMA` tables:*



为了描述不直接作为数据字典表上的VIEW实现的`INFORMATION_SCHEMA`查询，让我首先描述在`INFORMATION_SCHEMA`表中有两种类型的元数据：

1. **Static table metadata.** For example: `TABLE_SCHEMA`, `TABLE_NAME`, `TABLE_TYPE`, `ENGINE`. These statistics will be read directly from the data dictionary.
2. **Dynamic table metadata.** For example: `AUTO_INCREMENT`, `AVG_ROW_LENGTH`, `DATA_FREE`. Dynamic metadata frequently changes (for example: the auto_increment value will advance after each insert).In many cases the dynamic metadata will also incur some cost to accurately calculate on demand, and accuracy may not be beneficial for the typical query. Consider the case of the `DATA_FREE` statistic which shows the number of free bytes in a table – a cached value is usually sufficient.

 

1. 静态表元数据。例如：`TABLE_SCHEMA`, `TABLE_NAME`, `TABLE_TYPE`, `ENGINE`。这些静态数据将会从数据字典中直接读取
2. 动态表元数据。例如：`AUTO_INCREMENT`, `AVG_ROW_LENGTH`, `DATA_FREE`。动态元数据经常会变更(例如：自增值会在每次插入后自增)。在许多情况下，动态元数据也会产生一些成本，以便按需准确计算，并且对于某些特定的查询这个准确性并不会有用。考虑`DATA_FREE`统计信息的情况，该统计信息显示表中的空闲字节数 - 缓存值通常就足够了。

*In MySQL 8.0, the dynamic table metadata will **default to being cached**. This is configurable via the setting `information_schema_stats` (default `cached`), and can be changed to `information_schema_stats=latest` in order to **always** retrieve the dynamic information directly from the storage engine (at the cost of slightly higher query execution).*

在MySQL 8.0中，动态表元数据将**默认为缓存**。这可以通过设置`information_schema_stats`（默认`缓存`）进行配置，并且可以更改为`information_schema_stats = latest`，以便**始终**直接从存储引擎检索动态信息（以稍高的查询执行为代价）



*As an alternative, the user can also execute `ANALYZE TABLE` on the table, to update the cached dynamic statistics.*

作为替代方案，用户还可以在`表`上执行`ANALYZE TABLE`，以更新缓存的动态统计信息。



### Conclusion

The `INFORMATION_SCHEMA` design in 8.0 is a step forward enabling:

- - Simple and maintainable implementation.
  - Us to get rid of numerous `INFORMATION_SCHEMA` legacy bugs.
  - Proper use of the MySQL optimizer for `INFORMATION_SCHEMA` queries.
  - `INFORMATION_SCHEMA` queries to execute ~100 times faster, compared to 5.7, when retrieving static table metadata, as show in query Q1.



8.0中的`INFORMATION_SCHEMA`设计是向前迈出的一步：

- - 简单且可维护的实现。
  - 我们摆脱了很多的`INFORMATION_SCHEMA`遗留漏洞。
  - 正确使用MySQL优化器进行`INFORMATION_SCHEMA`查询。
  - 与检索静态表元数据时的5.7相比，`INFORMATION_SCHEMA`查询执行速度快~100倍，如查询Q1中所示。



*There is more to discuss about `INFORMATION_SCHEMA` in 8.0. The new implementation comes with a few changes in behavior when compared to the old `INFORMATION_SCHEMA` implementation. Please check the MySQL manual for more details about it.*

*Thanks for using MySQL!*



在8.0中还有更多关于`INFORMATION_SCHEMA的`讨论。与旧的`INFORMATION_SCHEMA`实现相比，新的实现方式有一些变化。有关它的更多详细信息，请查看MySQL手册。

感谢您使用MySQL！