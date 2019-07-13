---
title: InnoDB ALTER TABLE INDEX和INSERT之性能
tags: InnoDB
categories: InnoDB
---

> 原文作者：Satya Bodapati  
发布日期：2019年6月27日  
关键词： InnoDB, MySQL  
原文链接： https://www.percona.com/blog/2019/06/27/innodb-alter-table-add-index-and-insert-performance/   


<!-- more -->

在我的前一篇博客[InnoDB排序索引构建](https://www.percona.com/blog/2019/05/08/mysql-innodb-sorted-index-builds/)中，我解释了排序索引构建的内部处理过程。文章最后我说了“没有缺点”。

从MySQL 5.6开始，包括ALTER TABLE ADD INDEX在内的许多ddl都变成了“在线模式”。这意味着，当**ALTER操作**正在进行时，可以有并发的select和DMLs。请参阅MySQL文档[online DDL](https://dev.mysql.com/doc/refman/5.6/en/innodb-online-ddl-operations.html)。从文档中，我们可以看到，DDL操作 ***ALTER TABLE ADD INDEX*** 允许并发的DML。

在5.7中引入的排序索引构建的主要缺点是在进行更改时插入性能降低，在这篇文章中，我们特别讨论了正在执行 ***ALTER ADD INDEX*** 的表上的单线程插入性能。

如果表很大，比如大约有6亿行或更多行，插入甚至会导致服务器崩溃。对于运行数小时且并发插入等待超过600秒的更改尤其如此。InnoDB的监视器线程使服务器崩溃，声明插入等待latch超过600秒。它被报告为MySQL[Bug#82940](https://bugs.mysql.com/bug.php?id=82940)

## 它修复了吗？
这个问题从5.7 GA开始就存在了，并且在Percona Server for MySQL的最新版本 5.7.26-29和8.0.15-6中得到了修复，这是[PS-3410](https://jira.percona.com/browse/PS-3410) bug修复的一部分。完成插入的数量取决于表是压缩的还是未压缩的，以及页面大小。

Percona的补丁提供给了上游[Oracle MySQL](https://github.com/mysql/mysql-server/pull/268)，但是还没有包含其中。为了MySQL社区的利益，我们希望Oracle在下一个5.7版本中包含这个修复。

如果不能升级到PS-5.7.26，一个合理的解决方案是使用[pt-online-schema-change](https://www.percona.com/doc/percona-toolkit/LATEST/pt-online-schema-change.html)。使用此工具，确保磁盘空间至少等于原始表空间大小。

## 改善了多少？
改进的百分比取决于测试场景、机器配置等。请参阅下面的详细信息。

对于未压缩的表，修复版本(5.7.26)运行 ***ALTER ADD INDEX*** 时完成的插入比5.7.25(官方版)要多58%。

对于压缩表，当 ***ALTER ADD INDEX*** 运行时，完成的插入数要多322%。

## 与5.6相比如何?
修复之后，对于未压缩的表，***ALTER ADD INDEX***  期间(来自单个连接)完成的插入数与5.6相同。

对于压缩表，使用5.6.44完成的插入数比5.7.26(有一个固定值)多43%。这有点令人惊讶，需要做更多的分析才能找到原因。这是另一个话题。

## 从设计的角度来看问题
作为排序索引构建的一部分，索引正在构建时，index->lock获得X(排他)模式，此锁在已排序索引构建的整个过程中一直持有，是的，你读的没错，完整的信息见[PS-3410](https://jira.percona.com/browse/PS-3410)。

并发插入将能够看到正在构建一个“新索引”。对于这样的索引，插入到[在线ALTER日志](https://jira.percona.com/browse/PS-3410)中，稍后在ALTER末尾执行这些日志。作为此操作的一部分，INSERT尝试以S (shared)模式获取index->lock，以查看索引是否处于online或中止状态。

由于排序索引构建过程在整个过程中都以X模式持有index->lock，并发插入操作等待这个latch，如果索引很大，insert线程的等待时间将超过600秒，这会导致服务器崩溃。

## 修复  
解决方法相当简单。排序索引构建不需要在X模式下获取index->lock。在此阶段，未提交索引上没有并发读取。并发插入不会干扰已排序的索引构建。他们在线修改日志。因此，不获取正在构建的索引的index->lock是安全的。

## 测试用例

下面的MTR测试用例显示了运行ALTER时并发执行的insert的数量。注意，只有一个连接执行插入。

所有版本的测试都使用 ***innodb_buffer_pool_size = 1G*** 运行。使用了两个版本的表。一个具有常规16K页面大小，另一个具有4K页面大小的压缩表。

数据目录存储在RAM中，用于所有测试。您可以保存以下文件(例如mysql-test/t/ alter_insert_concurrent .test)，并运行MTR测试用例:
```bash
./mtr --mem main.alter_insert_concurrency --mysqld=--innodb_buffer_pool_size=1073741824
```

该测试向表插入1000万行，并创建索引(与 ***ALTER table t1 ADD INDEX*** 相同)，在另一个连接中，一个接一个地执行插入，直到ALTER完成。
```sql
--source include/have_innodb.inc
--source include/count_sessions.inc

connect (con1,localhost,root,,);

CREATE TABLE t1(
class INT,
id INT,
title VARCHAR(100),
title2 VARCHAR(100)
) ENGINE=InnoDB ROW_FORMAT=COMPRESSED KEY_BLOCK_SIZE=4;

DELIMITER |;
CREATE PROCEDURE populate_t1()
BEGIN
DECLARE i int DEFAULT 1;

START TRANSACTION;
WHILE (i <= 1000000) DO
INSERT INTO t1 VALUES (i, i, uuid(), uuid());
SET i = i + 1;
END WHILE;
COMMIT;
END|

CREATE PROCEDURE conc_insert_t1()
BEGIN
DECLARE i int DEFAULT 1;

SELECT COUNT(*) INTO @val FROM INFORMATION_SCHEMA.PROCESSLIST WHERE ID != CONNECTION_ID() AND info LIKE "CREATE INDEX%";

IF @val > 0 THEN
SELECT "ALTER STARTED";
END IF;

WHILE (@val > 0) DO
INSERT INTO t1 VALUES (i, i, uuid(), uuid());
SET i = i + 1;
SELECT COUNT(*) INTO @val FROM INFORMATION_SCHEMA.PROCESSLIST WHERE ID != CONNECTION_ID() AND info LIKE "CREATE INDEX%";
END WHILE;
SELECT concat('Total number of inserts is ', i);
END|
DELIMITER ;|

--disable_query_log
CALL populate_t1();
--enable_query_log

--connection con1
--send CREATE INDEX idx_title ON t1(title, title2);

--connection default
--sleep 1
--send CALL conc_insert_t1();

--connection con1
--reap

--connection default
--reap

--disconnect con1

DROP TABLE t1;

DROP PROCEDURE populate_t1;
DROP PROCEDURE conc_insert_t1;
--source include/wait_until_count_sessions.inc
```

## 测试数据
```bash
compressed 4k           : number of concurrent inserts (Avg of 6 runs)
==============            ============================
PS 5.7.25 (and earlier) : 2315
PS 5.7.26 (fix version) : 9785.66 (322% improvement compared to 5.7.25) (43% worse compared to 5.6)
PS 5.6                  : 17341


16K page size
=============
PS 5.7.25 (and earlier) : 3007
PS 5.7.26 (fix version) : 4768.33 (58.5% improvement compared to 5.7.25) (3.4% worse compared to 5.6)
PS 5.6                  : 4939
```