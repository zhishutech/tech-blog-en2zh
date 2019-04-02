MySQL InnoDB Cluster-一致性级别

一致性读取已经称为MySQL InnoDB Cluster使用者的关心的话题（看[这篇文章](https://lefred.be/content/mysql-group-replication-read-your-own-write-across-the-group/)和[这篇文章](https://proxysql.com/blog/proxysql-gtid-causal-reads)）

这就是为什么MySQL现在支持新的一致性模型在我们需要的时候来避免出现这种情况。

我强烈建议你阅读Nuno Carvalho和Anibal Pinto的系列文章

- [Group Replication – Consistency Levels](https://mysqlhighavailability.com/group-replication-consistency-levels/)
- [Group Replication: Preventing stale reads on primary fail-over!](https://mysqlhighavailability.com/group-replication-preventing-stale-reads-on-primary-fail-over/) (you can also check [this post](https://lefred.be/content/mysql-innodb-cluster-8-0-12-abort_server/))
- [Group Replication – Consistent Reads](https://mysqlhighavailability.com/group-replication-consistent-reads)
- [Group Replication – Consistent Reads Deep Dive](https://mysqlhighavailability.com/group-replication-consistent-reads-deep-dive/)

在看过这些优秀文章后，让我们通过一些例子来探究它是如何工作的。

## 环境

以下是环境设置：

- 3个成员：`mysql1`,`mysql2` & `mysql3`
- 集群运行模式为Single-Primary模式
- `mysql1`为Primary Master
- 预装了某些额外的sys views

## 案例1-EVENTUAL

`group_replication_consistency='EVENTUAL'`是默认的参数设置。以下是方案：

- 我们展示了控制Primary和其中一个Secondary间组复制一致性的session级变量的默认值
- 我们在Secondary Master(`mysql3`)锁定了一张表来阻塞应用来自Primay的事务
- 我们展示了即使我们在`mysql1`上提交了一个新的事务，我们在`mysql3`上读取表发现新的记录丢失了（由于锁的存在导致写入无法发生）
- 一旦释放锁，事务被应用并且在Secondary Master(`mysql3`)也变得可见了

## 案例2-BEFORE

在这个例子中，我们将会说明我们如何能够避免在Secondary Master上避免非一致性读取。

正如你所看到的，一旦我们设置了session变量控制一致性，在返回结果前，表上的操作（server设置了READ_ONLY）都会等待应用队列清空。

我们同样也可以注意到读取操作的等待时间（超时时间）非常长（默认8小时）并且可以修改到更短的时间：

我们设置了`wait_timeout=10`来定义到10秒

当达到超时时间后，将会报出如下错误：

`ERROR: 3797: Error while waiting for group transactions commit on group_replication_consistency= 'BEFORE'`

## 案例3-AFTER

只有当所有成员也应用了更改时，写入的机器才能接收到commit的返回。让我们来检查下这个动作：

这可以被视为同步写入，只有当所有的成员已经应用了，commit才能发生。 然而你可以发现在该一致性级别下，`wait_timeout`在写入中并没有影响。事实上，`wait_timeout`只会影响读取操作并且一致性级别不是`EVENTUAL`的情况下。

如果你不管任何原因锁定了一张表，这意味着这将引发几个问题。如果DBA需要做一些维护操作并且需要长时间锁定一张表，在维护过程中，必须避免在`AFTER`或`BEFORE_AND_AFTER`中查询。

## 案例4-Scope

在下面的视频中，我只想向您展示应用队列中"等待"事务的"范围"

我们将会再次锁定`t1`但是在一个Secondary master上，我们将会在表`t2`上执行一个`SELECT`，首先我们会设置`group_replication_consistency`为默认值（`EVENTUAL`）然后我们会修改一致性级别为`BEFORE`:

我们可以看到，只要它们是应用队列中的事务，如果你将一致性级别修改为`BEFORE`，它将队列中先前的事务被应用，即使这些表或记录是相关或不相关的，这都无所谓。

## 案例5-Observability

当然，我们需要检查发生了什么以及是否有查询在等待。

BEFORE

当`group_replication_consistency`设置为BEFORE（或者包含它），当一个事务正在等待应用队列被提交，可以通过以下sql来查询这些等待的事务：

```mysql
SELECT * FROM information_schema.processlist 
WHERE state='Executing hook on transaction begin.';
```

AFTER

当`group_replication_consistency`设置为AFTER（或者包含它），当一个事务正在等待在其他成员节点上被提交，可以通过以下sql来查询这些等待的事务：

```mysql
SELECT * FROM information_schema.processlist 
WHERE state='waiting for handler commit';
```

通过关联processlist和InnoDB Trx表能够得到更多信息

```mysql
SELECT *, TIME_TO_SEC(TIMEDIFF(now(),trx_started)) lock_time_sec 
FROM information_schema.innodb_trx JOIN information_schema.processlist
ON processlist.ID=innodb_trx.trx_mysql_thread_id 
WHERE state='waiting for handler commit' ORDER BY trx_started\G
```

## 结论

一致性级别是个非常好的特性，但如果在没有完全控制环境的情况下滥用，它可能会变得危险。

如果你没有完全控制你的环境请避免设置为全局`AFTER`。表锁、DDL、逻辑备份、快照等都可能在Primary Master上造成延迟提交和事务。但是，如果控制了你的环境，你现在完全可以控制你的MySQL InnoDB集群一致性级别。