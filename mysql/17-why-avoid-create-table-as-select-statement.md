## 为什么要避免使用“CREATE TABLE AS SELECT”语句

>作者： Alexander Rubin  
发布日期：2018-01-10  
关键词：create table as select, metadata locks, MySQL, open source database, row locking, table locking  
适用范围： Insight for DBAs, MySQL   
原文  http://www.percona.com/blog/2018/01/10/why-avoid-create-table-as-select-statement/


>In this blog post, I’ll provide an explanation why you should avoid using the CREATE TABLE AS SELECT statement.

在这篇博文中，我将解释为什么你应该避免使用CREATE TABLE AS SELECT语句。

>The SQL statement “create table <table_name> as select …” is used to create a normal or temporary table and materialize the result of the select. Some applications use this construct to create a copy of the table. This is one statement that will do all the work, so you do not need to create a table structure or use another statement to copy the structure.

SQL语句“create table <table_name> as select ...”用于创建普通表或临时表，并物化select的结果。某些应用程序使用这种结构来创建表的副本。一条语句完成所有工作，因此您无需创建表结构或使用其他语句来复制结构。

>At the same time there are a number of problems with this statement:
>1. You don’t create indexes for the new table
>2. You are mixing transactional and non-transactional statements in one transaction. As with any DDL, it will commit current and unfinished transactions
>3. CREATE TABLE … SELECT is not supported when using GTID-based replication
>4. Metadata locks won’t release until the statement is finished

与此同时，这种语句存在许多问题：

1. 您不为新表创建索引
2. 您在一个事务中混合了事务性和非事务性语句时，与任何DDL一样，它将提交当前和未完成的事务
3. 使用基于GTID的复制时不支持 CREATE TABLE ... SELECT
4. 在语句完成之前，元数据锁不会释放

### CREATE TABLE AS SELECT语句可以把事物变得很糟糕  

>Let’s imagine we need to transfer money from one account to another (classic example). But in addition to just transferring funds, we need to calculate fees. The developers decide to create a table to perform a complex calculation.

>Then the transaction looks like this:

让我们想象一下，我们需要将钱从一个账户转移到另一个账户（经典示例）。但除了转移资金外，我们还需要计算费用。开发人员决定创建一个表来执行复杂的计算。

然后事务看起来像这样：
```sql
begin;
update accounts set amount = amount - 100000 where account_id=123;
-- now we calculate fees
create table as select ... join ...
update accounts set amount = amount + 100000 where account_id=321;
commit;
```
>The “create table as select … join … ” commits a transaction that is not safe. In case of an error, the second account obviously will not be credited by the second account debit that has been already committed!

>Well, instead of “create table … “, we can use “create temporary table …” which fixes the issue, as temporary table creation is allowed.

“create table as select ... join ...”会提交一个事务，这是不安全的。如果出现错误，第二个帐户显然不会被已经提交的第二个帐户借记贷记！

好吧，我们可以使用“create temporary table …”来修复问题，而不是“create table … ”，因为允许临时表创建。

### GTID问题
>If you try to use CREATE TABLE AS SELECT when GTID is enabled (and ENFORCE_GTID_CONSISTENCY = 1) you get this error:

如果在启用GTID时尝试使用CREATE TABLE AS SELECT（并且ENFORCE_GTID_CONSISTENCY = 1），则会出现此错误：
```sql
General error: 1786 CREATE TABLE ... SELECT is forbidden when @@GLOBAL.ENFORCE_GTID_CONSISTENCY = 1.
```
>The application code may break.

应用程序代码可能会中断。

### 元数据锁问题
>Metadata lock issue for CREATE TABLE AS SELECT is less known. ([More information about the metadata locking in general](https://dev.mysql.com/doc/refman/5.7/en/metadata-locking.html)). Please note: MySQL metadata lock is different from InnoDB deadlock, row-level locking and table-level locking.

>This quick simulation demonstrates metadata lock:

CREATE TABLE AS SELECT的元数据锁定问题鲜为人知。（[有关元数据锁定的更多信息](https://dev.mysql.com/doc/refman/5.7/en/metadata-locking.html)）。 请注意：MySQL元数据锁与InnoDB死锁、行级锁、表级锁是不同的。

以下速模拟演示了元数据锁定：

**会话1:**
```
mysql> create table test2 as select * from test1;
```
**会话2:**
```
mysql> select * from test2 limit 10;
```
>-- blocked statement

语句被阻塞

>This statement is waiting for the metadata lock:

此语句正在等待元数据锁：

**会话3:**
```
mysql> show processlist;
+----+------+-----------+------+---------+------+---------------------------------+-------------------------------------------
| Id | User | Host      | db   | Command | Time | State                           | Info
+----+------+-----------+------+---------+------+---------------------------------+-------------------------------------------
|  2 | root | localhost | test | Query   |   18 | Sending data                    | create table test2 as select * from test1
|  3 | root | localhost | test | Query   |    7 | Waiting for table metadata lock | select * from test2 limit 10
|  4 | root | localhost | NULL | Query   |    0 | NULL                            | show processlist
+----+------+-----------+------+---------+------+---------------------------------+-------------------------------------------
```
>The same can happen another way: a slow select query can prevent some DDL operations (i.e., rename, drop, etc.):

同样地，可以采用另一种方式：慢查询可以阻塞某些DDL操作（即重命名，删除等）：
```
mysql> show processlistG
*************************** 1. row ***************************
           Id: 4
         User: root
         Host: localhost
           db: reporting_stage
      Command: Query
         Time: 0
        State: NULL
         Info: show processlist
    Rows_sent: 0
Rows_examined: 0
    Rows_read: 0
*************************** 2. row ***************************
           Id: 5
         User: root
         Host: localhost
           db: test
      Command: Query
         Time: 9
        State: Copying to tmp table
         Info: select count(*), name from test2 group by name order by cid
    Rows_sent: 0
Rows_examined: 0
    Rows_read: 0
*************************** 3. row ***************************
           Id: 6
         User: root
         Host: localhost
           db: test
      Command: Query
         Time: 5
        State: Waiting for table metadata lock
         Info: rename table test2 to test4
    Rows_sent: 0
Rows_examined: 0
    Rows_read: 0
3 rows in set (0.00 sec)
```
>As we can see, CREATE TABLE AS SELECT can affect other queries. However, the problem here is not the metadata lock itself (the metadata lock is needed to preserve consistency). The problem is that the
***metadata lock will not be released until the statement is finished***.

我们可以看到，CREATE TABLE AS SELECT可以影响其他查询。但是，这里的问题不是元数据锁本身（需要元数据锁来保持一致性）。问题是 ***在语句完成之前不会释放元数据锁***。

>The fix is simple: copy the table structure first by doing “create table new_table like old_table”, then do “insert into new_table select …”. The metadata lock is still held for the create table part (very short), but isn’t for the “insert … select” part (the total time to hold the lock is much shorter). To illustrate the difference, let’s look at two cases:

>1. With “create table table_new as select … from table1“, other application connections can’t read from the destination table (table_new) for the duration of the statement (even “show fields from table_new” will be blocked)
>2. With “create table new_table like old_table” + “insert into new_table select …”, other application connections can’t read from the destination table during the “insert into new_table select …” part.


修复很简单：首先复制表结构，执行“ create table new_table like old_table”，然后执行“insert into new_table select ...”。元数据锁仍然在创建表部分（非常短）持有，但“insert … select”部分不会持有（保持锁定的总时间要短得多）。为了说明不同之处，让我们看看以下两种情况：
1. 使用“create table table_new as select ... from table1 ”，其他应用程序连接 在语句的持续时间内 无法读取目标表（table_new）（甚至“show fields from table_new”将被阻塞）
2. 使用“create table new_table like old_table”+“insert into new_table select ...”，在“insert into new_table select ...”这部分期间，其他应用程序连接无法读取目标表。

>In some cases, however, the table structure is not known beforehand. For example, we may need to materialize the result set of a complex select statement, involving joins and/or group by. In this case, we can use this trick:

然而，在某些情况下，表结构事先是未知的。例如，我们可能需要物化复杂select语句的结果集，包括joins、and/or、group by。在这种情况下，我们可以使用这个技巧：
```
create table new_table as select ... join ... group by ... limit 0;
insert into new_table as select ... join ... group by ...
```

>The first statement creates a table structure and doesn’t insert any rows (LIMIT 0). The first statement places a metadata lock. However, it is very quick. The second statement actually inserts rows into the table and doesn’t place a metadata lock.

第一个语句创建一个表结构，不插入任何行（LIMIT 0）。第一个语句持有元数据锁。但是，它非常快。第二个语句实际上是在表中插入行，而不持有元数据锁。













