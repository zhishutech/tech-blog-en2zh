
# Four Ways MySQL Executes GROUP BY
# MySQL的四种GROUP BY用法
原文：[Four Ways MySQL Executes GROUP BY](https://www.percona.com/blog/2018/02/05/four-ways-to-execute-mysql-group-by/)
译者：魏新平

------
在本文当中，我将介绍MySQL执行GROUP BY的四种方法。
>In this blog post, I’ll look into four ways MySQL executes GROUP BY. 

在我的上一篇文章当中，我们知道了通过索引或者其他的方式获取数据可能不是语句执行最耗时的操作。比如，MySQL 的GROUP BY可能会占据语句执行时间的90%.
>In my previous blog post, we learned that indexes or other means of finding data might not be the most expensive part of query execution. For example, MySQL GROUP BY could potentially be responsible for 90% or more of the query execution time. 

当MySQL执行group by的时候，最复杂的操作就是聚合计算。想具体了解算法的可以看这里[UDF Aggregate Functions](https://dev.mysql.com/doc/refman/5.7/en/udf-aggr-calling.html)。简单的说，UDF函数会一个接着一个的获取构成单个组的所有行，这样就可以在处理下个组之前，计算出当前组的聚合值。
>The main complexity when MySQL executes GROUP BY is computing aggregate functions in a GROUP BY statement. How this works is shown in the documentation for UDF Aggregate Functions. As we see, the requirement is that UDF functions get all values that constitute the single group one after another. That way, it can compute the aggregate function value for the single group before moving to another group.

这样处理有一个明显的问题。在大多数情况下，源数据并不是根据GROUP BY的组顺序进行保存的。我们需要特殊的步骤去处理MySQL的GROUP BY.
>The problem, of course, is that in most cases the source data values aren’t grouped. Values coming from a variety of groups follow one another during processing. As such, we need a special step to handle MySQL GROUP BY.

让我们再来看一下上一篇博客当中例子的表
>Let’s look at the same table we looked at before:

```Shell
mysql> show create table tbl G
*************************** 1. row ***************************
      Table: tbl
Create Table: CREATE TABLE `tbl` (
 `id` int(11) NOT NULL AUTO_INCREMENT,
 `k` int(11) NOT NULL DEFAULT '0',
 `g` int(10) unsigned NOT NULL,
 PRIMARY KEY (`id`),
 KEY `k` (`k`)
) ENGINE=InnoDB AUTO_INCREMENT=2340933 DEFAULT CHARSET=latin1
1 row in set (0.00 sec)
```

下面就开始介绍MySQL执行group by的4种方法
>And the same GROUP BY statements executed in different ways:

1:利用索引排序进行GROUP BY操作
>1: Index Ordered GROUP BY in MySQL

```Shell
mysql> select k, count(*) c from tbl group by k order by k limit 5;
+---+---+
| k | c |
+---+---+
| 2 | 3 |
| 4 | 1 |
| 5 | 2 |
| 8 | 1 |
| 9 | 1 |
+---+---+
5 rows in set (0.00 sec)
mysql> explain select k, count(*) c from tbl group by k order by k limit 5 G
*************************** 1. row ***************************
          id: 1
 select_type: SIMPLE
       table: tbl
  partitions: NULL
        type: index
possible_keys: k
         key: k
     key_len: 4
         ref: NULL
        rows: 5
    filtered: 100.00
       Extra: Using index
1 row in set, 1 warning (0.00 sec)
```
在该查询当中，GROUP BY字段上面有索引，就可以在扫描索引数据的同时进行聚合计算。（索引上，数据是按照组来排序的）
>In this case, we have an index on the column we use for GROUP BY. This way, we can just scan data group by group and perform GROUP BY on the fly (inexpensively).

当我们使用limit来限制组的数量或者当覆盖索引被使用的时候，语句执行效率会特别高，因为只对索引进行顺序扫描是非常快速的操作。
>It works especially well when we use LIMIT to restrict the number of groups we retrieve or when a “covering index” is in use, as a sequential index-only scan is a very fast operation.

有时候，虽然聚合的组数很少，但是由于没有使用到覆盖索引，索引顺序扫描会造成大量的IO。所以这个方式可能不是最优的。
>If you have a small number of groups though, and no covering index, index order scans can cause a lot of IO. So this might not be the most optimal plan.

2:利用的External Sort方式执行GROUP BY
>2: External Sort GROUP BY in MySQL
```Shell
mysql> explain select SQL_BIG_RESULT g, count(*) c from tbl group by g limit 5 G
*************************** 1. row ***************************
          id: 1
 select_type: SIMPLE
       table: tbl
  partitions: NULL
        type: ALL
possible_keys: NULL
         key: NULL
     key_len: NULL
         ref: NULL
        rows: 998490
    filtered: 100.00
       Extra: Using filesort
1 row in set, 1 warning (0.00 sec)
mysql> select SQL_BIG_RESULT g, count(*) c from tbl group by g limit 5;
+---+---+
| g | c |
+---+---+
| 0 | 1 |
| 1 | 2 |
| 4 | 1 |
| 5 | 1 |
| 6 | 2 |
+---+---+
5 rows in set (0.88 sec)
```
如果我们没有一个允许我们按照组顺序扫描数据的索引，我们可以通过外部排序来排序数据（在MySQL中也被称为“filesort”）。
>If we do not have an index that allows us to scan the data in group order, we can instead get data sorted through an external sort (also referred to as “filesort” in MySQL).

你可能注意到我是用了SQL_BIG_RESULT来获取这个执行计划。没有它，MySQL不会选择这个执行计划。
>You may notice I’m using an SQL_BIG_RESULT hint here to get this plan. Without it, MySQL won’t choose this plan in this case.

一般来说，只有当我们需要处理大量的组数时，MySQL才会使用这个计划，因为在这种情况下，外排序比用临时表处理更有效率（我们将在下面讨论）。
>In general, MySQL prefers to use this plan only if we have a large number of groups, because in this case sorting is more efficient than having a temporary table (which we will talk about next).

3：利用临时表执行GROUP BY
>3: Temporary Table GROUP BY in MySQL

```Shell
mysql> explain select  g, sum(g) s from tbl group by g limit 5 G
*************************** 1. row ***************************
          id: 1
 select_type: SIMPLE
       table: tbl
  partitions: NULL
        type: ALL
possible_keys: NULL
         key: NULL
     key_len: NULL
         ref: NULL
        rows: 998490
    filtered: 100.00
       Extra: Using temporary
1 row in set, 1 warning (0.00 sec)
mysql> select  g, sum(g) s from tbl group by g order by null limit 5;
+---+------+
| g | s    |
+---+------+
| 0 |    0 |
| 1 |    2 |
| 4 |    4 |
| 5 |    5 |
| 6 |   12 |
+---+------+
5 rows in set (7.75 sec)
```
在这种情况下，MySQL 也会进行全表扫描。但不会进行额外的排序，而是会创建一张临时表。该临时表当中每一个组会先保存一行记录，在处理剩余的行的时候，会把对应的行更新到临时表当中。 但如果结果表太大，更新可能会导致大量的磁盘IO。在这种情况下，外部排序会更有优势。
>In this case, MySQL also does a full table scan. But instead of running additional sort passes, it creates a temporary table instead. This temporary table contains one row per group, and with each incoming row the value for the corresponding group is updated. Lots of updates! While this might be reasonable in-memory, it becomes very expensive if the resulting table is so large that updates are going to cause a lot of disk IO. In this case, external sort plans are usually better.

请注意，虽然MySQL在此用例中默认选择了此计划，但如果我们不提供任何hint，它将比使用SQL_BIG_RESULT hint的计划慢10倍。
>Note that while MySQL selects this plan by default for this use case, if we do not supply any hints it is almost 10x slower than the plan we get using the SQL_BIG_RESULT hint.

你可能会注意到我添加了“ORDER BY NULL”.这是为了让执行计划只使用临时表进行GROUP BY操作。不然我们会得到其他的执行计划。
>You may notice I added “ORDER BY NULL” to this query. This is to show you “clean” the temporary table only plan. Without it, we get this plan:

```Shell
mysql> explain select  g, sum(g) s from tbl group by g limit 5 G
*************************** 1. row ***************************
          id: 1
 select_type: SIMPLE
       table: tbl
  partitions: NULL
        type: ALL
possible_keys: NULL
         key: NULL
     key_len: NULL
         ref: NULL
        rows: 998490
    filtered: 100.00
       Extra: Using temporary; Using filesort
1 row in set, 1 warning (0.00 sec)
```
上面的执行计划当中，同时使用了临时表和文件排序，这样的结果是非常糟糕的。
>In it, we get the “worst of both worlds” with Using Temporary Table and filesort.  

MySQL 5.7 总是会对GROUP BY的结果按照组的顺序进行排序，即使语句并没有要求他这么做。ORDER BY NULL 可以取消这种默认排序。
>MySQL 5.7 always returns GROUP BY results sorted in group order, even if this the query doesn’t require it (which can then require an expensive additional sort pass). ORDER BY NULL signals the application doesn’t need this.

在某些情况下， 比如使用集合函数访问不同表中的列的JOIN查询，使用临时表可能是处理GROUP BY的唯一选择。
>You should note that in some cases – such as JOIN queries with aggregate functions accessing columns from different tables – using temporary tables for GROUP BY might be the only option.

假如想强制MySQL使用临时表处理GROUP BY，可以使用SQL_SMALL_RESULT  hint。
>If you want to force MySQL to use a plan that does temporary tables for GROUP BY, you can use the SQL_SMALL_RESULT  hint.

### 4：利用索引Skip-Scan-Based的方式进行group by
>4:  Index Skip-Scan-Based GROUP BY in MySQL

前面3种GROUP BY的 执行方式适用于所有的聚合函数。但是有些聚合函数会使用第四种方法。
>The previous three GROUP BY execution methods apply to all aggregate functions. Some of them, however, have a fourth method.

```Shell
mysql> explain select k,max(id) from tbl group by k G
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
        rows: 2
    filtered: 100.00
       Extra: Using index for group-by
1 row in set, 1 warning (0.00 sec)
mysql> select k,max(id) from tbl group by k;
+---+---------+
| k | max(id) |
+---+---------+
| 0 | 2340920 |
| 1 | 2340916 |
| 2 | 2340932 |
| 3 | 2340928 |
| 4 | 2340924 |
+---+---------+
5 rows in set (0.00 sec)
```
这种方式只适用于MIN()和MAX()。再有索引的情况下，求最大最小值不需要对所有的值进行处理。
>This method applies only to very special aggregate functions: MIN() and MAX(). These do not really need to go through all the rows in the group to compute the value at all.

MySQL可以直接查到最大最小值（假如有索引的话）
>They can just jump to the minimum or maximum group value in the group directly (if there is such an index).

如何直接的获取最大的ID值呢，如果索引是创建在k列上？这是InnoDB表。记住InnoDB会把所有主键值扩展到其他索引上面。（k）变成了（k，ID），允许我们使用Skip-Scan来优化这个语句。
>How can you find MAX(ID) value for each group if the index is only built on (K) column? This is an InnoDB table. Remember InnoDB tables effectively append the PRIMARY KEY to all indexes. (K) becomes (K,ID), allowing us to use Skip-Scan optimization for this query.

这种优化方式只有在每个组有大量数据的情况下才会生效。否则，MySQL更偏向于常规的方式来执行这个语句（就像文章开头第一种方法）
>This optimization is only enabled if there is a large number of rows per group. Otherwise, MySQL prefers more conventional means to execute this query (like Index Ordered GROUP BY detailed in approach #1).

MIN()/MAX()还有其他的优化方式。比如，在没有GROUP BY的情况下使用聚合函数（整张表就是一个组），MySQL在统计分析阶段就从索引中获取这些值，避免了在执行阶段读取表。
>While we’re on MIN()/MAX() aggregate functions, other optimizations apply to them as well. For example, if you have an aggregate function with no GROUP BY (effectively  having one group for all tables), MySQL fetches those values from indexes during a statistics analyzes phase and avoids reading tables during the execution stage altogether:

```Shell
mysql> explain select max(k) from tbl G
*************************** 1. row ***************************
          id: 1
 select_type: SIMPLE
       table: NULL
  partitions: NULL
        type: NULL
possible_keys: NULL
         key: NULL
     key_len: NULL
         ref: NULL
        rows: NULL
    filtered: NULL
       Extra: Select tables optimized away
1 row in set, 1 warning (0.00 sec)
```
### Filtering和Group By
>Filtering and Group By

上面已经介绍了4种MySQL执行GROUP BY的方式。为了简单，我直接对整张表进行了GROUP BY，并没有过滤任何数据。如果使用了WHERE进行数据过滤，上面方法还是适用的。
>We have looked at four ways MySQL executes GROUP BY.  For simplicity, I used GROUP BY on the whole table, with no filtering applied. The same concepts apply when you have a WHERE clause:

```Shell
mysql> explain select  g, sum(g) s from tbl where k>4 group by g order by NULL limit 5 G
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
        rows: 1
    filtered: 100.00
       Extra: Using index condition; Using temporary
1 row in set, 1 warning (0.00 sec)
```
在这个例子当中，我们使用了k列索引的范围扫描来过滤数据，在有临时表的时候进行group by操作。
>For this case, we use the range on the K column for data filtering/lookup and do a GROUP BY when there is a temporary table.

有些情况，这些方法并不冲突。但其他的情况下，我们必须二选一。要么用索引来GROUP BY，要么用来过滤数据。
>In some cases, the methods do not conflict. In others, however, we have to choose either to use one index for GROUP BY or another index for filtering:

```Shell
mysql> alter table tbl add key(g);
Query OK, 0 rows affected (4.17 sec)
Records: 0  Duplicates: 0  Warnings: 0
mysql> explain select  g, sum(g) s from tbl where k>1 group by g limit 5 G
*************************** 1. row ***************************
          id: 1
 select_type: SIMPLE
       table: tbl
  partitions: NULL
        type: index
possible_keys: k,g
         key: g
     key_len: 4
         ref: NULL
        rows: 16
    filtered: 50.00
       Extra: Using where
1 row in set, 1 warning (0.00 sec)
mysql> explain select  g, sum(g) s from tbl where k>4 group by g limit 5 G
*************************** 1. row ***************************
          id: 1
 select_type: SIMPLE
       table: tbl
  partitions: NULL
        type: range
possible_keys: k,g
         key: k
     key_len: 4
         ref: NULL
        rows: 1
    filtered: 100.00
       Extra: Using index condition; Using temporary; Using filesort
1 row in set, 1 warning (0.00 sec)
```
根据这个查询中k列使用的特定常量，我们可以看到，我们要么使用g列索引进行group by（放弃使用k列索引快速的过滤数据），要么使用k列索引进行数据过滤（使用临时表来处理GROUP BY），没办法同时使用到两个索引。
>Depending on specific constants used in this query, we can see that we either use an index ordered scan for GROUP BY (and  “give up”  benefiting from the index to resolve the WHERE clause), or use an index to resolve the WHERE clause (but use a temporary table to resolve GROUP BY).

根据我的经验，MySQL在这种情况下可能无法做出正确的选择。那时就需要使用FORCE INDEX hint来让语句按照你想要的方式执行。
>In my experience, this is where MySQL GROUP BY does not always make the right choice. You might need to use FORCE INDEX to execute queries the way you want them to.

我希望这篇文章可以让大家更好的理解MySQL执行GROUP BY的方法。在下一篇博客当中，我将介绍一些优化group by的方法。
>I hope this article provides a good overview of how MySQL executes GROUP BY. In my next blog post, we will look into techniques you can use to optimize GROUP BY queries.
