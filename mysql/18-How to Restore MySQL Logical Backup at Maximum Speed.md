## 如何最快恢复逻辑备份
>作者： Nickolay Ihalainen  
发布日期：2018-01-10  
关键词： backup, data integrity, InnoDB, MySQL logical backups, restore database  
适用范围： Backups, Database Monitoring, Hardware and Storage, InnoDB, Insight for DBAs, MySQL   
原文  https://www.percona.com/blog/2018/02/22/restore-mysql-logical-backup-maximum-speed/


The ability to restore MySQL logical backups is a significant part of disaster recovery procedures. It’s a last line of defense.

恢复MySQL逻辑备份的能力是灾难恢复过程的重要部分。这也是最后一道防线。

Even if you lost all data from a production server, physical backups (data files snapshot created with an offline copy or with Percona XtraBackup) could show the same internal database structure corruption as in production data. Backups in a simple plain text format allow you to avoid such corruptions and migrate between database formats (e.g., during a software upgrade and downgrade), or even help with migration from completely different database solution.

即使生产服务器的所有数据都丢失了，物理备份(使用冷备份或Percona XtraBackup)可能出现与生产数据相同的内部数据库结构损坏。你可以通过简单纯文本格式的备份来避免这种损坏，并且可以在数据库格式之间进行迁移(例如，在软件升级和降级期间)，甚至可以用于完全不同的数据库之间迁移。

Unfortunately, the restore speed for logical backups is usually bad, and for a big database it could require days or even weeks to get data back. Thus it’s important to tune backups and MySQL for the fastest data restore and change settings back before production operations.

不幸的是，逻辑备份的恢复速度通常很糟糕，对于大型数据库，可能需要几天甚至几周才能恢复数据。因此，优化备份和MySQL以获得最快的数据恢复速度是非常重要的，然后在生产操作之前再把设置更改回来。

## Disclaimer
## 免责声明
All results are specific to my combination of hardware and dataset, but could be used as an illustration for MySQL database tuning procedures related to logical backup restore.

所有结果都是基于我的硬件和数据集，但可以作为与逻辑备份恢复相关，MySQL数据库调优过程的一个例子。

## Benchmark
## 基准测试
There is no general advice for tuning a MySQL database for a bulk logical backup load, and any parameter should be verified with a test on your hardware and database. In this article, we will explore some variables that help that process. To illustrate the tuning procedure, I’ve downloaded IMDB CSV files and created a MySQL database with pyimdb.

对于批量逻辑备份导入MySQL数据库的调优，没有统一的建议，任何参数的修改都应该基于你的硬件和数据库进行测试和验证。在本文中，我们将探讨一些有助于这一过程的参数。为了演示调优过程，我下载了IMDB CSV文件，并使用pyimdb创建了一个MySQL数据库。

You may repeat the whole benchmark procedure, or just look at settings changed and resulting times.

您可以重复整个基准测试过程，或者只查看设置的更改和耗时。

Database:

- 16GB – InnoDB database size
- 6.6GB – uncompressed mysqldump sql
- 5.8GB – uncompressed CSV + create table statements.

数据库：
- 16 G - innodb 数据库大小
- 6.6 G - 无压缩的 mysqldump sql
- 5.8 G - 无压缩的CSV + 创建表语句

The simplest restore procedure for logical backups created by the mysqldump tool:

mysqldump逻辑备份最简单的恢复过程如下:

```
mysql -e 'create database imdb;'
time mysql imdb < imdb.sql
# real 129m51.389s
```
This requires slightly more than two hours to restore the backup into the MySQL instance started with default settings.

使用默认设置将备份恢复到MySQL实例需要两个多小时。

I’m using the Docker image percona:latest – it contains Percona Server 5.7.20-19 running on a laptop with 16GB RAM, Intel(R) Core(TM) i7-7700HQ CPU @ 2.80GHz, two disks: SSD KINGSTON RBU-SNS and HDD HGST HTS721010A9.

我使用的是Docker image percona:latest——它包含了运行在16GB RAM、Intel(R) Core(TM) i7-7700HQ CPU @ 2.80GHz笔记本电脑上的percona Server 5.7.20-19，以及两个磁盘:SSD KINGSTON rm - sns和HDD HGST HTS721010A9。

Let’s start with some “good” settings: buffer pool bigger than default, 2x1GB transaction log files, disable sync (because we are using slow HDD), and set big values for IO capacity,
the load should be faster with big batches thus use 1GB for max_allowed_packet.

让我们从一些“好用的”设置开始: 将缓冲池大小调来大于默认值，2x1GB事务日志文件，禁用sync (因为我们使用的是低速HDD)，并调大IO capacity 的值，设置max_allowed_packet为1GB加速大批量加载。

Values were chosen to be bigger than the default MySQL parameters because I’m trying to see the difference between the usually suggested values (like 80% of RAM should belong to InnoDB buffer pool).

之所以将这些MySQL参数的默认值调大，是因为我想看看与常规值之间的区别(比如将 InnoDB buffer pool设置为机器总内存的80%)。


```
docker run --publish-all --name p57 -it -e MYSQL_ALLOW_EMPTY_PASSWORD=1 percona:5.7 
  --innodb_buffer_pool_size=4GB 
  --innodb_log_file_size=1G 
  --skip-log-bin 
  --innodb_flush_log_at_trx_commit=0 
  --innodb_flush_method=nosync 
  --innodb_io_capacity=2000 
  --innodb_io_capacity_max=3000 
  --max_allowed_packet=1G
  time (mysql --max_allowed_packet=1G imdb1 < imdb.sql )
  # real 59m34.252s
```
The load is IO bounded, and there is no reaction on set global foreign_key_checks=0 and unique_checks=0 because these variables are already disabled in the dump file.

IO限制加载效率，设置set global foreign_key_check =0和unique_check =0，并没有效果，因为这些变量在备份文件中已经被禁用了。

How can we reduce IO?

那么该怎样减少IO呢?

Disable InnoDB double write: --innodb_doublewrite=0

禁用InnoDB double write:——innodb_doublewrite=0


```
time (mysql --max_allowed_packet=1G imdb1 < imdb.sql )
# real 44m49.963s
```
A huge improvement, but we still have an IO-bounded load.

改善了许多，但是仍然有负载IO限制.

We will not be able to improve load time significantly for IO bounded load. Let’s move to SSD:

对于IO负载限制，我们无法显著提高加载时间。让我们转到SSD

```
time (mysql --max_allowed_packet=1G imdb1 < imdb.sql )
# real 33m36.975s
```
Is it vital to disable disk sync for the InnoDB transaction log?

禁用磁盘同步是否对于InnoDB事务日志来说是否重要?

```
sudo rm -rf mysql/*
docker rm p57
docker run -v /home/ihanick/Private/Src/tmp/data-movies/imdb.sql:/root/imdb.sql -v /home/ihanick/Private/Src/tmp/data-movies/mysql:/var/lib/mysql 
--name p57 -it -e MYSQL_ALLOW_EMPTY_PASSWORD=1 percona:5.7 
--innodb_buffer_pool_size=4GB 
--innodb_log_file_size=1G 
--skip-log-bin 
--innodb_flush_log_at_trx_commit=0 
--innodb_io_capacity=700 
--innodb_io_capacity_max=1500 
--max_allowed_packet=1G 
--innodb_doublewrite=0
# real 33m49.724s
```
There is no significant difference.

无显著性差异。

By default, mysqldump produces SQL data, but it could also save data to CSV format:

默认情况下，mysqldump会生成SQL数据，但也可以将数据保存为CSV格式

```
cd /var/lib/mysql-files
mkdir imdb
chown mysql:mysql imdb/
time mysqldump --max_allowed_packet=128M --tab /var/lib/mysql-files/imdb imdb1
# real 1m45.983s
sudo rm -rf mysql/*
docker rm p57
docker run -v /srv/ihanick/tmp/imdb:/var/lib/mysql-files/imdb -v /home/ihanick/Private/Src/tmp/data-movies/mysql:/var/lib/mysql
--name p57 -it -e MYSQL_ALLOW_EMPTY_PASSWORD=1 percona:5.7
--innodb_buffer_pool_size=4GB
--innodb_log_file_size=1G
--skip-log-bin
--innodb_flush_log_at_trx_commit=0
--innodb_io_capacity=700
--innodb_io_capacity_max=1500
--max_allowed_packet=1G
--innodb_doublewrite=0
time (
mysql -e 'drop database imdb1;create database imdb1;set global FOREIGN_KEY_CHECKS=0;'
(echo "SET FOREIGN_KEY_CHECKS=0;";cat *.sql) | mysql imdb1 ;
for i in $PWD/*.txt ; do mysqlimport imdb1 $i ; done
)
# real 21m56.049s
1.5X faster, just because of changing the format from SQL to CSV!
```

We’re still using only one CPU core, let’s improve the load with the –use-threads=4 option:

我们仍然只使用一个CPU core，让我们使用-use-threads =4选项来提高负载:

```
time (
mysql -e 'drop database if exists imdb1;create database imdb1;set global FOREIGN_KEY_CHECKS=0;'
(echo "SET FOREIGN_KEY_CHECKS=0;";cat *.sql) | mysql imdb1
mysqlimport --use-threads=4 imdb1 $PWD/*.txt
)
# real 15m38.147s
```
In the end, the load is still not fully parallel due to a big table: all other tables are loaded, but one thread is still active.

最后，由于一个大表，负载仍然不是完全并行的：所有其他表都已经加载了，但是有一个线程仍然是活动的。

Let’s split CSV files into smaller ones. For example, 100k rows in each file and load with GNU/parallel:

让我们把CSV文件分割成更小的文件。例如，每个文件中有100k行，用GNU/parallel加载:

```
# /var/lib/mysql-files/imdb/test-restore.sh
apt-get update ; apt-get install -y parallel
cd /var/lib/mysql-files/imdb
time (
cd split1
for i in ../*.txt ; do echo $i ; split -a 6 -l 100000 -- $i `basename $i .txt`. ; done
for i in `ls *.*|sed 's/^[^.]+.//'|sort -u` ; do
mkdir ../split-$i
for j in *.$i ; do mv $j ../split-$i/${j/$i/txt} ; done
done
)
# real 2m26.566s
time (
mysql -e 'drop database if exists imdb1;create database imdb1;set global FOREIGN_KEY_CHECKS=0;'
(echo "SET FOREIGN_KEY_CHECKS=0;";cat *.sql) | mysql imdb1
parallel 'mysqlimport imdb1 /var/lib/mysql-files/imdb/{}/*.txt' ::: split-*
)
#real 16m50.314s
```
Split is not free, but you can split your dump files right after backup.

分割并不是没有时间开销，但是您可以在备份之后立即分割文件。

The load is parallel now, but the single big table strikes back with ‘setting auto-inc lock’ in SHOW ENGINE INNODB STATUSG

现在的负载是并行的，但是在SHOW ENGINE INNODB STATUSG时，单个的大表仍然会显示 ‘setting auto-inc lock’。

Using the --innodb_autoinc_lock_mode=2 option fixes this issue: 16m2.567s.

可以使用--innodb_autoinc_lock_mode=2选项 解决这个问题:16m2.567s.

We got slightly better results with just mysqlimport --use-threads=4. Let’s check if hyperthreading helps and if the problem caused by “parallel” tool:

- Using four parallel jobs for load: 17m3.662s
- Using four parallel jobs for load and two threads: 16m4.218s

使用mysqlimport --use-threads=4，我们得到了稍微好一点的结果。让我们检查一下超线程是否有帮助，以及由“并行”工具引起的问题:
- 使用4个并行作业进行加载:17m3.662s
- 使用四个并行作业进行加载和两个线程:16m4.218s

There is no difference between GNU/Parallel and --use-threads option of mysqlimport.

使用GNU/Parallel和mysqlimport的--use-threads选项并没有区别。

==Why 100k rows? With 500k rows: 15m33.258s==

一次100万行会怎样？只有16m 52.357s

Now we have performance better than for mysqlimport --use-threads=4.

现在我们有了比mysqlimport--use-threads=4更好的性能.

How about 1M rows at once? Just 16m52.357s.

如果一次执行100万行呢？只需要16m52.357s

I see periodic flushing logs message with bigger transaction logs (2x4GB): 12m18.160s

我看到较大事务日志(2x4GB)的周期刷新日志消息时间：12m18.160s

```
--innodb_buffer_pool_size=4GB --innodb_log_file_size=4G --skip-log-bin --innodb_flush_log_at_trx_commit=0 --innodb_io_capacity=700 --innodb_io_capacity_max=1500 --max_allowed_packet=1G --innodb_doublewrite=0 --innodb_autoinc_lock_mode=2 --performance-schema=0
```
Let’s compare the number with myloader 0.6.1 also running with four threads (myloader have only -d parameter, myloader execution time is under corresponding mydumper command):

我们将这个数字与myloader 0.6.1进行比较，myloader也有4个线程在运行(myloader只有-d参数，myloader的执行时间在mydumper命令下面):

```
# oversized statement size to get 0.5M rows in one statement, single statement per chunk file
mydumper -B imdb1 --no-locks --rows 500000 --statement-size 536870912 -o 500kRows512MBstatement
17m59.866s
mydumper -B imdb1 --no-locks -o default_options
17m15.175s
mydumper -B imdb1 --no-locks --chunk-filesize 128 -o chunk128MB
16m36.878s
mydumper -B imdb1 --no-locks --chunk-filesize 64 -o chunk64MB
18m15.266s
```
It will be great to test mydumper with CSV format, but unfortunately, it wasn’t implemented in the last 1.5 years: https://bugs.launchpad.net/mydumper/+bug/1640550.

用CSV格式测试mydumper会很好，但不幸的是，它在过去1年半没有更新:https://bugs.launchpad.net/mydumper/+bug/1640550

Returning back to parallel CSV files load, even bigger transaction logs 2x8GB: 11m15.132s.

回到并行CSV文件加载，甚至更大的事务日志2x8GB: 11m15.132s。

What about a bigger buffer pool: --innodb_buffer_pool_size=12G? 9m41.519s

那么更大的buffer pool呢: --innodb_buffer_pool_size=12G? 9m41.519s

Let’s check six-year-old server-grade hardware: Intel(R) Xeon(R) CPU E5-2430 with SAS raid (used only for single SQL file restore test) and NVMe (Intel Corporation PCIe Data Center SSD, used for all other tests).

检查一下6年的服务器硬件: 有SAS raid的Intel(R) Xeon(R) CPU E5-2430(仅用于单个SQL文件恢复测试)和NVMe (Intel Corporation PCIe Data Center SSD，用于所有其他测试)。

I’m using similar options as for previous tests, with 100k rows split for CSV files load:

前面的测试我都使用类似的参数设置，加载10万行分割的CSV文件:

```
--innodb_buffer_pool_size=8GB --innodb_log_file_size=8G --skip-log-bin --innodb_flush_log_at_trx_commit=0 --innodb_io_capacity=700 --innodb_io_capacity_max=1500 --max_allowed_packet=1G --innodb_doublewrite=0 --innodb_autoinc_lock_mode=2
```
- Single SQL file created by mysqldump loaded for 117m29.062s = 2x slower.
- 24 parallel processes of mysqlimport: 11m51.718s
- Again hyperthreading making a huge difference! 12 parallel jobs: 18m3.699s.
- Due to higher concurrency, adaptive hash index is a reason for locking contention. After disabling it with --skip-innodb_adaptive_hash_index: 10m52.788s.
- In many places, disable unique checks referred as a performance booster: 10m52.489s
- You can spend more time reading advice about unique_checks, but it might help for some databases with many unique indexes (in addition to primary one).
- The buffer pool is smaller than the dataset, can you change old/new pages split to make insert faster? No: --innodb_old_blocks_pct=5 : 10m59.517s.
- O_DIRECT is also recommended: --innodb_flush_method=O_DIRECT: 11m1.742s.
- O_DIRECT is not able to improve performance by itself, but if you can use a bigger buffer pool: O_DIRECT + 30% bigger buffer pool: --innodb_buffeer_pool_size=11G: 10m46.716s.


- 加载mysqldump创建的单个SQL文件耗时117m29.062s要慢2倍。
- 24个并行进程的mysqlimport:11m51.718s
- 超线程带来了巨大的不同!12个并行作业:18m3.699s。
- 由于更高的并发性，自适应哈希索引是锁争用的原因之一。禁用它之后--skip-innodb_adaptive_hash_index耗时: 10m52.788s。
- 在许多情况下，禁用唯一性检查意味着性能风暴:10m52.489s。
- 您可以多花时间阅读一下有关unique_checks的建议，但是对于一些有许多惟一索引(除了主索引之外)的数据库，它可能会有所帮助。
- 如果buffer pool比数据集小，你能改变旧/新页面分割使插入更快吗?不能:--innodb_old_blocks_pct=5: 10m59.517s。
- 还建议使用O_DIRECT：--innodb_flush_method=O_DIRECT: 11m1.742s.
- O_DIRECT本身无法提高性能，但如果可以使用更大的buffer pool:O_DIRECT + 增大30%的buffer pool:--innodb_buffeer_pool_size=11G: 10m46.716s。

## Conclusions
## 结论

- There is no common solution to improve logical backup restore procedure.
- If you have IO-bounded restore: disable InnoDB double write. It’s safe because even if the database crashes during restore, you can restart the operation.
- Do not use SQL dumps for databases > 5-10GB. CSV files are much faster for mysqldump+mysql. Implement mysqldump --tabs+mysqlimport or use mydumper/myloader with appropriate chunk-filesize.
- The number of rows per load data infile batch is important. Usually 100K-1M, use binary search (2-3 iterations) to find a good value for your dataset.
- InnoDB log file size and buffer pool size are really important options for backup restore performance.
- O_DIRECT reduces insert speed, but it’s good if you can increase the buffer pool size.
- If you have enough RAM or SSD, the restore procedure is limited by CPU. Use a faster CPU (higher frequency, turboboost).
- Hyperthreading also counts.
- A powerful server could be slower than your laptop (12×2.4GHz vs. 4×2.8+turboboost).
- Even with modern hardware, it’s hard to expect backup restore faster than 50MBps (for the final size of InnoDB database).
- You can find a lot of different advice on how to improve backup load speed. Unfortunately, it’s not possible to implement improvements blindly, and you should know the limits of your system with general Unix performance tools like vmstat, iostat and various MySQL commands like SHOW ENGINE INNODB STATUS (all can be collected together with pt-stalk).
- Percona Monitoring and Management (PMM) also provides good graphs, but you should be careful with QAN: full slow query log during logical database dump restore can cause significant processing load.
- Default MySQL settings could cost you 10x backup restore slowdown
- This benchmark is aimed at speeding up the restore procedure while the application is not running and the server is not used in production. Make sure that you have reverted all configuration parameters back to production values after load. For example, if you disable the InnoDB double write buffer during restore and left it enabled in production, you may have scary data corruption due to partial InnoDB pages writes.
- If the application is running during restore, in most cases you will get an inconsistent database due to missing support for locking or correct transactions for restore methods (discussed above).
 
- 没有统一的解决方案来提升逻辑备份恢复的效率。
- 如果你有IO限制的恢复:禁用InnoDB double write。这是安全的，因为即使数据库在恢复期间崩溃，您也可以重新开始操作。
- 不要对大于5-10GB的数据库使用SQL dumps。CSV文件要比mysqldump+mysql快得多。执行mysqldump --tabs+mysqlimport，或者使用具有合适chunk-filesize的mydumper/myloader。
- 每次批量加载数量的行数很重要。通常是10万-100万，使用二分查找(2-3次迭代)来为数据集找到一个合适的值。
- InnoDB log file size和 buffer pool size是备份恢复性能的重要选项。
- O_DIRECT会降低插入速度，但如果可以调大buffer pool size，这将会是一个好的选择。
- 如果RAM或SSD的资源足够，恢复过程就只会受到CPU的限制。 建议使用更快的CPU(更高的频率，睿频加速)。
- 超线程也很重要
- 一个性能高的服务器可能会比你的笔记本电脑慢(12×2.4 ghz和4×2.8 + turboboost)。
- 即使使用现代的硬件，备份恢复速度超过50MBps的期望也很难达到(对于InnoDB数据库的最终速度大小)
- 你可以找到很多不同的建议去如何提高备份加载速度，不幸的是，盲目地实施改进是不可能的成功的，您可以使用常用的Unix性能分析工具如mstat、iostat等以及SHOW ENGINE INNODB STATUS等各种MySQL命令来获取你的系统限制(这些信息都可以使用 pt-stalk收集)。
- Percona Monitoring and Management (PMM)也提供了许多图表，但是您应该小心QAN:在逻辑数据库备份恢复期间产生的大量慢日志可能会导致严重的负载。
- 默认MySQL设置会使你的备份恢复速度减慢10倍
- 此基准测试的目的是加快备份恢复过程，在应用程序不运行且服务器没有用在生产环境下的时候。记得确保在数据加载完成后已将所有的配置参数生产中的值。例如，如果在恢复期间禁用InnoDB double write buffer，并在生产环境中继续禁用它，可能会由于InnoDB页面的部分写入而导致严重的数据损坏。
- 如果应用程序在恢复期间运行，在大多数情况下，由于缺少对锁或恢复方法的正确事务的支持，您将获得不一致的数据库(如上所述)。
