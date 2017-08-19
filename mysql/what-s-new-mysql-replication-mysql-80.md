## [What’s New With MySQL Replication in MySQL 8.0](https://severalnines.com/blog/what-s-new-mysql-replication-mysql-80)

MySQL 8.0, which as of now (August 2017) is still in beta state, brings some nice improvements to replication. Originally, it was developed for Group Replication (GR), but as GR uses regular replication under the hood, “normal” MySQL replication benefited from it. The improvement we mentioned is dependency tracking information stored in the binary log. What happens is that MySQL 8.0 now has a way to store information about which rows were affected by a given transaction (so called writeset), and it compares writesets from different transactions. This makes it possible to identify those transactions which did not work on the same subset of rows and, therefore, these may be applied in parallel. This may allow to increase the parallelization level by several times compared to the implementation from MySQL 5.7. What you need to keep in mind is that, eventually, a slave will see a different view of the data, one that never appeared on the master. This is because transactions may be applied in a different order than on the master. This should not be a problem though. The current implementation of multithreaded replication in MySQL 5.7 may also cause this issue unless you explicitly enable slave-preserve-commit-order.

To control this new behavior, a variable binlog_transaction_dependency_tracking has been introduced. It can take three values:

COMMIT_ORDER: this is the default one, it uses the default mechanism available in MySQL 5.7.
WRITESET: It enables better parallelization and the master starts to store writeset data in binary log.
WRITESET_SESSION: This ensures that transactions will be executed on the slave in order and the issue with a slave that sees a state of database which never was seen on the master is eliminated. It reduces parallelization but it still can provide better throughput than the default settings.
Benchmark

In July, on mysqlhighavailability.com, Vitor Oliveira wrote a post where he tried to measure the performance of new modes. He used the best case scenario - no durability whatsoever, to showcase the difference between old and new modes. We decided to use the same approach, this time in a more real-world setup: binary log enabled with log_slave_updates. Durability settings were left to default (so, sync_binlog=1 - that’s new default in MySQL 8.0, doublewrite buffer enabled, InnoDB checksums enabled etc.) Only exception in durability was innodb_flush_log_at_trx_commit set to 2.

We used m4.2xl instances, 32G, 8 cores (so slave_parallel_workers was set to 8). We also used sysbench, oltp_read_write.lua script. 16 million rows in 32 tables were stored on 1000GB gp2 volume (that’s 3000 IOPS). We tested the performance of all of the modes for 1, 2, 4, 8, 16 and 32 concurrent sysbench connections. Process was as follows: stop slave, execute 100k transactions, start slave and calculate how long it takes to clear the slave lag.

![](https://severalnines.com/sites/default/files/blog/node_5091/image1.jpg)

First of all, we don’t really know what happened when sysbench was executed using 1 thread only. Each test was executed five times after a warmup run. This particular configuration was tested two times - results are stable: single-threaded workload was the fastest. We will be looking into it further to understand what happened.

Other than that, the rest of the results are in line with what we expected. COMMIT_ORDER is the slowest one, especially for low traffic, 2-8 threads. WRITESET_SESSION performs typically better than COMMIT_ORDER but it’s slower than WRITESET for low-concurrent traffic.

How it can help me?

The first advantage is obvious: if your workload is on the slow side yet your slaves have tendency to fall back in replication, they can benefit from improved replication performance as soon as the master will be upgraded to 8.0. Two notes here: first - this feature is backward compatible and 5.7 slaves can also benefit from it. Second - a reminder that 8.0 is still in beta state, we don’t encourage you to use beta software on production, although in dire need, this is an option to test. This feature can help you not only when your slaves are lagging. They may be fully caught up but when you create a new slave or reprovision existing one, that slave will be lagging. Having the ability to use “WRITESET” mode will make the process of provisioning a new host much faster.

All in all, this feature will have much bigger impact that you may think. Given all of the benchmarks showing regressions in performance when MySQL handles traffic of low concurrency, anything which can help to speed up the replication in such environments is a huge improvement.

If you use intermediate masters, this is also a feature to look for. Any intermediate master adds some serialization into how transactions are handled and executed - in real world, the workload on an intermediate master will almost always be less parallel than on the master. Utilizing writesets to allow better parallelization not only improves parallelization on the intermediate master but it also can improve parallelization on all of its slaves. It is even possible (although it would require serious testing to verify all pieces will fit correctly) to use an 8.0 intermediate master to improve replication performance of your slaves (please keep in mind that MySQL 5.7 slave can understand writeset data and use it even though it cannot generate it on its own). Of course, replicating from 8.0 to 5.7 sounds quite tricky (and it’s not only because 8.0 is still beta). Under some circumstances, this may work and can speed up CPU utilization on your 5.7 slaves.

Other changes in MySQL replication

Introducing writesets, while it is the most interesting, it is not the only change that happened to MySQL replication in MySQL 8.0. Let’s go through some other, also important changes. If you happen to use a master older than MySQL 5.0, 8.0 won’t support its binary log format. We don’t expect to see many such setups, but if you use some very old MySQL with replication, it’s definitely a time to upgrade.

Default values have changed to make sure that replication is as crash-safe as possible: master_info_repository and relay_log_info_repository are set to TABLE. Expire_log_days has also been changed - now the default value is 30. In addition to expire_log_days, a new variable has been added, binlog_expire_log_seconds, which allows for more fine-grained binlog rotation policy. Some additional timestamps have been added to the binary log to improve observability of replication lag, introducing microsecond granularity.

By all means, this is not a full list of changes and features related to MySQL replication. If you’d like to learn more, you can check the MySQL changelogs. Make sure you reviewed all of them - so far, features have been added in all 8.0 versions.

As you can see, MySQL replication is still changing and becoming better. As we said at the beginning, it has to be a slow-paced process but it’s really great to see what is ahead. It’s also nice to see the work for Group Replication trickling down and reused in the “regular” MySQL replication.
