> 原文 ：<https://lefred.be/content/mysql-innodb-cluster-consistency-levels/>
>
> 作者：[lefred](https://lefred.be/content/author/lefred/)
>
> 翻译：无名队

*Consistency during reads have been a small concern from the adopters of MySQL InnoDB Cluster (see [this post](https://lefred.be/content/mysql-group-replication-read-your-own-write-across-the-group/) and [this one](https://proxysql.com/blog/proxysql-gtid-causal-reads)).*

*This is why MySQL supports now (since 8.0.14) a new consistency model to avoid such situation when needed.*

*[Nuno Carvalho](https://twitter.com/rekconk) and Aníbal Pinto already posted a blog series I highly encourage you to read:*

- *[Group Replication – Consistency Levels](https://mysqlhighavailability.com/group-replication-consistency-levels/)*
- *[Group Replication: Preventing stale reads on primary fail-over!](https://mysqlhighavailability.com/group-replication-preventing-stale-reads-on-primary-fail-over/) (you can also check [this post](https://lefred.be/content/mysql-innodb-cluster-8-0-12-abort_server/))*
- *[Group Replication – Consistent Reads](https://mysqlhighavailability.com/group-replication-consistent-reads)*
- *[Group Replication – Consistent Reads Deep Dive](https://mysqlhighavailability.com/group-replication-consistent-reads-deep-dive/)*

*After those great articles, let’s check how that does work with some examples.*

一致性读取已经成为MySQL InnoDB Cluster使用者的关心的话题（看[这篇文章](https://lefred.be/content/mysql-group-replication-read-your-own-write-across-the-group/)和[这篇文章](https://proxysql.com/blog/proxysql-gtid-causal-reads)）

这就是为什么MySQL 8.0.14开始，当我们需要避免这些情况时，支持使用新的一致性模型。

我强烈建议你阅读Nuno Carvalho和Anibal Pinto的系列文章

- [Group Replication – Consistency Levels](https://mysqlhighavailability.com/group-replication-consistency-levels/)
- [Group Replication: Preventing stale reads on primary fail-over!](https://mysqlhighavailability.com/group-replication-preventing-stale-reads-on-primary-fail-over/) (you can also check [this post](https://lefred.be/content/mysql-innodb-cluster-8-0-12-abort_server/))
- [Group Replication – Consistent Reads](https://mysqlhighavailability.com/group-replication-consistent-reads)
- [Group Replication – Consistent Reads Deep Dive](https://mysqlhighavailability.com/group-replication-consistent-reads-deep-dive/)

在看过这些优秀文章后，让我们通过一些例子来探究它是如何工作的。

## 环境

*This is how the environment is setup:*

- *3 members: `mysql1`, `mysql2` & `mysql3`*
- *the cluster runs in Single-Primay mode*
- *`mysql1` is the Primary Master*
- *some [extra sys views](https://gist.github.com/lefred/153448f7ea0341d6d0daa2738db6fcd8) are installed*

以下是环境设置：

- 3个成员：`mysql1`,`mysql2` & `mysql3`
- 集群运行模式为Single-Primary模式
- `mysql1`为Primary Master
- 预装了某些额外的sys views

## 案例1-EVENTUAL

*This is the default behavior (`group_replication_consistency='EVENTUAL'`). The scenario is the following:*

- *we display the default value of the session variable controlling the Group Replication Consistency on the Primary and on one Secondary*
- *we lock a table on a Secondary master (`mysql3`) to block the apply of the transaction coming from the Primary*
- *we demonstrate that even if we commit a new transaction on `mysql1`, we can read the table on `mysql3` and the new record is missing (the write could not happen due to the lock)*
- *once unlocked, the transaction is applied and the record is visible on the Secondary master (`mysql3`) too*

group_replication_consistency='EVENTUAL'`是默认的参数设置。以下是方案：

- 我们演示了控制Primary和其中一个Secondary间组复制一致性的session级变量的默认值
- 我们在Secondary Master(`mysql3`)锁定了一张表来阻塞应用来自Primay的事务
- 我们演示了即使我们在`mysql1`上提交了一个新的事务，我们在`mysql3`上读取表发现新的记录丢失了（由于锁的存在导致写入无法发生）
- 一旦释放锁，事务被应用并且在Secondary Master(`mysql3`)也变得可见了

## 案例2-BEFORE

*In this example, we will illustrate how we can avoid inconsistent reads on a Secondary master:*

在这个例子中，我们将会说明我们如何在Secondary Master上避免非一致性读取。

*As you could notice, once we have set the session variable controlling the consistency, operations on the table (the server is READ-ONLY) are waiting for the Apply Queue to be empty before returning the result set.*

正如你所看到的，一旦我们设置了session变量控制一致性，在返回结果前，表上的操作（server设置了READ_ONLY）都会等待应用队列清空。

*We could also notice that the wait time (timeout) for this read operation is very long (8 hours by default) and can be modified to a shorter period:*

我们同样也可以注意到读取操作的等待时间（超时时间）非常长（默认8小时）并且可以修改到更短的时间：

*We used `SET wait_timeout=10` to define it to 10 seconds.*

我们设置了`wait_timeout=10`来定义到10秒

*When the timeout is reached, the following error is returned:*

当达到超时时间后，将会报出如下错误：

```mysql
ERROR: 3797: Error while waiting for group transactions commit on group_replication_consistency= 'BEFORE'
```

## 案例3-AFTER

*It’s also possible to return from commit on the **writer** only when all members applied the change too. Let’s check this in action too:*

只有当所有成员也应用了更改时，写入的机器才能接收到commit的返回。让我们来检查下这个动作：

*This can be considered as synchronous writes as the return from commit happens only when all members have applied it. However you could also notice that in this consistency level, `wait_timeout` has not effect on the write. In fact `wait_timeout` has only effect on **read operations** when the consistency level is different than `EVENTUAL`.*

这可以被视为同步写入，只有当所有的成员已经应用了，commit才能发生。 然而你可以发现在该一致性级别下，`wait_timeout`在写入中并没有影响。事实上，`wait_timeout`只会影响读取操作并且一致性级别不是`EVENTUAL`的情况下。

*This means that this can lead to several issues if you lock a table for any reason. If the DBA needs to perform some maintenance operations and requires to lock a table for a long time, it’s mandatory to not operate queries in `AFTER` or ` BEFORE_AND_AFTER`while in such maintenance.*

如果你不管任何原因锁定了一张表，这意味着这将引发几个问题。如果DBA需要做一些维护操作并且需要长时间锁定一张表，在维护过程中，必须避免在`AFTER`或`BEFORE_AND_AFTER`中查询。

## 案例4-Scope

*In the following video, I just want to show you the “scope” of these “waits” for transactions that are in the applying queue.*

在下面的视频中，我只想向您演示应用队列中"等待"事务的"范围"

*We will lock again `t1` but on a Secondary master, we will perform a `SELECT` from table `t2`, the first time we will keep the default value of `group_replication_consistency`(`EVENTUAL`) and the second time we will change the consistency level to `BEFORE` :*

我们将会在一个Secondary master上再次锁定`t1`表，然后在`t2`表上执行一个select，第一次测试我们会设置`group_replication_consistency`为默认值（`EVENTUAL`），第二次测试我们会调整一致性级别为`BEFORE`。

*We could see that as soon as they are transactions in the apply queue, if you change the consistency level to something `BEFORE`, it needs to wait for the previous transactions in the queue to be applied even if those events are related or not to the same table(s) or record(s). It doesn’t matter*

我们可以看到，只要它们是应用队列中的事务，如果你将一致性级别修改为`BEFORE`，它将队列中先前的事务被应用，即使这些表或记录是相关或不相关的，这都无所谓。

## 案例5-Observability

*Of course it’s possible to check what’s going on and if queries are waiting for something.*

当然，我们需要检查发生了什么以及是否有查询在等待。

BEFORE

*When `group_replication_consistency` is set to **BEFORE** (or includes it), while a transaction is waiting for the applying queue to be committed, it’s possible to track those waiting transactions by running the following query:*

当`group_replication_consistency`设置为BEFORE（或者包含它），当一个事务正在等待应用队列被提交，可以通过以下sql来查询这些等待的事务：

```mysql
SELECT * FROM information_schema.processlist 
WHERE state='Executing hook on transaction begin.';
```

AFTER

*When `group_replication_consistency` is set to **AFTER** (or includes it), while a transaction is waiting for the transaction to be committed on the other members too, it’s possible to track those waiting transactions by running the following query:*

当`group_replication_consistency`设置为AFTER（或者包含它），当一个事务正在等待在其他成员节点上被提交，可以通过以下sql来查询这些等待的事务：

```mysql
SELECT * FROM information_schema.processlist 
WHERE state='waiting for handler commit';
```

*It’s also possible to have even more information joining the processlist and InnoDB Trx tables:*

通过关联processlist和InnoDB Trx表能够得到更多信息

```mysql
SELECT *, TIME_TO_SEC(TIMEDIFF(now(),trx_started)) lock_time_sec 
FROM information_schema.innodb_trx JOIN information_schema.processlist
ON processlist.ID=innodb_trx.trx_mysql_thread_id 
WHERE state='waiting for handler commit' ORDER BY trx_started\G
```

## 结论

*This consistency level is a wonderful feature but it could become dangerous if abused without full control of your environment.*

一致性级别是个非常好的特性，但如果在没有完全控制环境的情况下滥用，它可能会变得危险。

*I would avoid to set anything `AFTER` globally if you don’t control completely your environment. Table locks, DDLs, logical backups, snapshots could all delay the commits and transactions could start pilling up on the Primary Master. But if you control your environment, you have now the complete freedom to control completely the consistency you need on your MySQL InnoDB Cluster.*

如果你没有完全控制你的环境请避免设置为全局`AFTER`。表锁、DDL、逻辑备份、快照等都可能在Primary Master上造成延迟提交和事务。但是，如果控制了你的环境，你现在完全可以控制你的MySQL InnoDB集群一致性级别。