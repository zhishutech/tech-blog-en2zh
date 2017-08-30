# [MySQL 8.0 中复制的新特性](https://github.com/zhishutech/tech-blog-en2zh/blob/master/mysql/what-s-new-mysql-replication-mysql-80.md)

截止目前（2017年8月），MySQL 8.0 仍然是 beta 测试版本，它为复制带来了一些非常棒的改进。最初，这些改进是为组复制（GR）开发的，但由于 GR 在底层使用常规复制，所以传统的 MySQL 复制也能由此获益。我们这里提到的改进是存储在二进制日志中的依赖关系跟踪信息。MySQL 8.0 使用某种方法来存储那些受事务影响的行的相关信息（这些行被称为`writeset`），且它会比较来自不同事务的`writesets`。这样就能识别出那些操作行没有交集的事务，这些事务可以在从库上被并行地回放。与 MySQL 5.7 的实现相比，这也许能增加数倍的并行化程度。值得注意的是，最终，从库将会出现从未在主库上出现过的数据视图。这是因为事务可能被按照与主库不同的顺序去回放。当然，这其实没有什么问题。目前在 MySQL 5.7 中实现的多线程复制也可能会导致这个问题，除非您明确地启用 `slave-preserve-commit-order` 参数。
>MySQL 8.0, which as of now (August 2017) is still in beta state, brings some nice improvements to replication. Originally, it was developed for Group Replication (GR), but as GR uses regular replication under the hood, “normal” MySQL replication benefited from it. The improvement we mentioned is dependency tracking information stored in the binary log. 
What happens is that MySQL 8.0 now has a way to store information about which rows were affected by a given transaction (so called writeset), and it compares writesets from different transactions. This makes it possible to identify those transactions which did not work on the same subset of rows and, therefore, these may be applied in parallel. 
This may allow to increase the parallelization level by several times compared to the implementation from MySQL 5.7. What you need to keep in mind is that, eventually, a slave will see a different view of the data, one that never appeared on the master. 
This is because transactions may be applied in a different order than on the master. This should not be a problem though. The current implementation of multithreaded replication in MySQL 5.7 may also cause this issue unless you explicitly enable slave-preserve-commit-order.


为了控制这个新的行为，MySQL 引入了一个新的变量：`binlog_transaction_dependency_tracking`。 它可以取三个值：

* COMMIT_ORDER：默认值，它使用 MySQL 5.7 中可用的默认机制。
* WRITESET：它能实现更好的并行化，并且在主库的二进制日志中存储写集数据。
* WRITE_SESSION：它确保事务在从库中按顺序执行，并且消除了从库中看到主库从未出现过的数据库状态的问题。它降低了并行化程度，但是仍然提供了比默认设置更好吞吐量。

>To control this new behavior, a variable `binlog_transaction_dependency_tracking` has been introduced. It can take three values:  
>
>COMMIT\_ORDER: this is the default one, it uses the default mechanism available in MySQL 5.7.  
>
>WRITESET: It enables better parallelization and the master starts to store writeset data in binary log.  
>
>WRITESET\_SESSION: This ensures that transactions will be executed on the slave in order and the issue with a slave that sees a state of database which never was seen on the master is eliminated. It reduces parallelization but it still can provide better throughput than the default settings.


### 基准测试

7月份，在 mysqlhighavailability.com 网站上，Vitor Oliveira 写了一篇文章分析了他对新模式进行性能测试的情况。他使用 MySQL 中性能最好的情况 —— 无持久性设置，展示了新旧模式之间的区别。我们决定使用相同的方法进行测试，但同时设置一个更加真实的配置，启用 `log_slave_updates` 参数使之开启二进制日志记录。关于持久性设置，除了将 `innodb_flush_log_at_trx_commit` 设置为 2 ，其他均保留默认值（所以，`sync_binlog`=1 —— 这是 MySQL 8.0 中的新默认值，启用 `doublewrite buffer`，启用 `InnoDB checksums` ，等等）。
>In July, on mysqlhighavailability.com, Vitor Oliveira wrote a post where he tried to measure the performance of new modes.He used the best case scenario - no durability whatsoever, to showcase the difference between old and new modes.We decided to use the same approach, this time in a more real-world setup: binary log enabled with `log_slave_updates`. Durability settings were left to default (so, `sync_binlog`=1 - that’s new default in MySQL 8.0, doublewrite buffer enabled, InnoDB checksums enabled etc.) Only exception in durability was `innodb_flush_log_at_trx_commit` set to 2.


我们使用 m4.2xl 实例，32G，8核（所以参数 `slave_parallel_workers` 也设置为 8）。我们同样使用 `sysbench`，`oltp_read_write.lua` 脚本。32 张表中的 1600 万行数据，存储在 1000GB gp2 卷（即 3000 IOPS）上。我们测试了所有模式下 1、2、4、8、16 和 32 个并发 `sysbench` 连接的性能，过程如下：关闭从库，执行 10 万个事务，启动从库并计算清除从库延迟所需的时间。
>We used m4.2xl instances, 32G, 8 cores (so `slave_parallel_workers` was set to 8). We also used sysbench, `oltp_read_write.lua` script. 16 million rows in 32 tables were stored on 1000GB gp2 volume (that’s 3000 IOPS). We tested the performance of all of the modes for 1, 2, 4, 8, 16 and 32 concurrent sysbench connections. Process was as follows: stop slave, execute 100k transactions, start slave and calculate how long it takes to clear the slave lag.

![](https://severalnines.com/sites/default/files/blog/node_5091/image1.jpg)


首先，我们其实并不清楚使用单线程的 `sysbench` 压测时数据库到底发生了什么。每一次测试我们都在给数据库预热后，再执行了5次。这个特殊参数配置我们测试了2次，结果值是稳定一致的。使用单线程的 `sysbench` 压测是最快的。我们将会进一步研究，以了解发生了什么。￼
>First of all, we don’t really know what happened when sysbench was executed using 1 thread only. Each test was executed five times after a warmup run. This particular configuration was tested two times - results are stable: single-threaded workload was the fastest. We will be looking into it further to understand what happened.


除此之外，其余的结果都符合我们的预期。`COMMIT_ORDER` 是最慢的，特别是低流量的时候（2-8线程）。`WRITESET_SESSION` 通常比 `COMMIT_ORDER` 更好，但是对于低并发流量，它比 `WRITESET` 慢。
>Other than that, the rest of the results are in line with what we expected. `COMMIT_ORDER` is the slowest one, especially for low traffic, 2-8 threads. `WRITESET_SESSION` performs typically better than `COMMIT_ORDER` but it’s slower than `WRITESET` for low-concurrent traffic.


### 报告信息总结

第一个优点是显而易见的：如果主库工作负载较低，且从库复制速度倾向于变慢，那么只要你将主库升级到 8.0 就能从其改进的复制性能中获益。
>The first advantage is obvious: if your workload is on the slow side yet your slaves have tendency to fall back in replication, they can benefit from improved replication performance as soon as the master will be upgraded to 8.0.


这里有两个注意事项：

1. 这个特性是向后兼容的，所以 5.7 的从库也能从中获益；
2. 请注意 MySQL 8.0 依然是 beta 测试版本，我们不鼓励您在生产环境中使用测试版软件，尽管你非常需要这些新功能。

>Two notes here: first - this feature is backward compatible and 5.7 slaves can also benefit from it. Second - a reminder that 8.0 is still in beta state, we don’t encourage you to use beta software on production, although in dire need, this is an option to test.


这个特性不仅在你从库延迟时有作用，在你创建一个全新的从库或者重新配置一个已有的从库时，它也完全能跟得上。有了使用 "WRITESET" 模式的能力，配置一个新主机的过程将会变得更快。
>This feature can help you not only when your slaves are lagging. They may be fully caught up but when you create a new slave or reprovision existing one, that slave will be lagging. Having the ability to use “WRITESET” mode will make the process of provisioning a new host much faster.


总而言之，这个特性带来的影响可能会产生超乎你想象。鉴于所有基准测试显示当 MySQL 处理低并发流量时性能较差，任何有助于加速在这种环境中复制的改进都将是巨大的进步。
>All in all, this feature will have much bigger impact that you may think. Given all of the benchmarks showing regressions in performance when MySQL handles traffic of low concurrency, anything which can help to speed up the replication in such environments is a huge improvement.


如果你使用中继主库，这同样是你需要的特性。任何复制架构的中继主库都会在处理和执行事务时增加一些串行化信息。 —— 在真实的生产环境中，中继主库的工作负载几乎都是比主库的并行化程度低。利用 `writesets` 来实现更好的并行化，不仅能提高中继主库的并行化程度，而且可以提高所有从库的并行化程度。甚至可以使用 8.0 中继主库（虽然要经过严格的测试以验证所有的功能正常使用）来提高从库（请注意 MySQL 5.7 从库虽然不能自己生成 `writeset` 数据，但是它能识别和使用 `writeset` 数据）的复制性能。当然，从 8.0 复制到 5.7 听起来有点诡异（不仅仅是因为 8.0 还是 beta 版本）。在某些情况下，这可能会起作用，并可以加快 5.7 从库上的 CPU 利用率。
>If you use intermediate masters, this is also a feature to look for.Any intermediate master adds some serialization into how transactions are handled and executed . - in real world, the workload on an intermediate master will almost always be less parallel than on the master.Utilizing writesets to allow better parallelization not only improves parallelization on the intermediate master but it also can improve parallelization on all of its slaves.It is even possible (although it would require serious testing to verify all pieces will fit correctly) to use an 8.0 intermediate master to improve replication performance of your slaves (please keep in mind that MySQL 5.7 slave can understand writeset data and use it even though it cannot generate it on its own).Of course, replicating from 8.0 to 5.7 sounds quite tricky (and it’s not only because 8.0 is still beta). Under some circumstances, this may work and can speed up CPU utilization on your 5.7 slaves.


### MySQL 复制的其他变化

除了最有趣的 `writesets` 新特性，MySQL 8.0 中关于 MySQL 复制的其他变化也是值得关注的。我们来看看其他的一些重要变化。如果你碰巧使用了一个比 5.0 版本还老的MySQL，请注意 MySQL 8.0 不再支持它的二进制日志格式。尽管我们不期望看到许多类似上述的 MySQL 复制配置，但是如果你真的在复制架构中使用一些非常老的 MySQL 版本，那真的是时候去升级了。
>Introducing writesets, while it is the most interesting, it is not the only change that happened to MySQL replication in MySQL 8.0. Let’s go through some other, also important changes. If you happen to use a master older than MySQL 5.0, 8.0 won’t support its binary log format. We don’t expect to see many such setups, but if you use some very old MySQL with replication, it’s definitely a time to upgrade.


为了尽可能的保证复制架构中的MySQL数据库崩溃恢复时的数据库的安全性，MySQL 8.0 中一些默认值已更改：`master_info_repository` 和 `relay_log_info_repository` 默认设置为 TABLE。`expire_log_days` 的默认值也变成了 30 。除了 `expire_log_days` 之外，还添加了一个新的参数 `binlog_expire_log_seconds`，它允许更细粒度的 binlog 轮换策略。二进制日志中添加了一些额外的时间戳，使复制延迟时可以更好地被观察，同时还引入了微秒级别的粒度。
>Default values have changed to make sure that replication is as crash-safe as possible: `master_info_repository` and `relay_log_info_repository` are set to TABLE. `Expire_log_days` has also been changed - now the default value is 30. 
In addition to `expire_log_days`, a new variable has been added, `binlog_expire_log_seconds`, which allows for more fine-grained binlog rotation policy. Some additional timestamps have been added to the binary log to improve observability of replication lag, introducing microsecond granularity.


然而，这并不是与 MySQL 复制相关的更改和功能的完整列表。如果你想了解更多信息，可以查看 MySQL 更新日志。以确保您已经查看到所有相关信息。 —— 到目前为止，所有 8.0 版本都添加了这些特性。
>By all means, this is not a full list of changes and features related to MySQL replication. If you’d like to learn more, you can check the MySQL changelogs. Make sure you reviewed all of them - so far, features have been added in all 8.0 versions.


正如你所看到的，MySQL 复制仍然在变化而且越来越好。正如我们刚才所说的那样，MySQL 复制变化是一个缓慢的进程，但是 MySQL 复制的前景是非常好的。我们很高兴看到组复制的工作成果能在常规的 MySQL 复制中使用并且用得很好。
>As you can see, MySQL replication is still changing and becoming better. As we said at the beginning, it has to be a slow-paced process but it’s really great to see what is ahead. It’s also nice to see the work for Group Replication trickling down and reused in the “regular” MySQL replication.
