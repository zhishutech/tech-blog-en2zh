>作者：Alexey Stroganov, Laurynas Biveinis and Vadim Tkachenko 
>发布时间：2016.3.17  
>文章关键字：Benchmarks,MySQL,Percona Server  

在这篇文章中，我们将讨论Percona Server 5.7有哪些性能提升。
从Percona Server 5.6发布以来，我们引入了几个重要的更新，有助于解决高并发I/O负载场景下的性能问题。我们的一些研究和改进在MySQL 5.7 —— 目前最好的MySQL版本之一被重现了。但即使MySQL 5.7在扩展性和性能等方便都有所提升，我们还是发现了可以增进I/O工作负载性能的一些地方。

Percona Server 5.7.11 在这方面有两个主要的性能特性：
**`多线程LRU刷新`**.这个特性在Percona Server 5.6就存在了，但当时作用有限。我们从page clean线程中分离了LRU刷新线程，现在完全只做flush list的刷新。除了其他几个重要的改变，这显著提升了I/O绑定的工作负载性能。MySQL 5.7更进一步引入了page clean 池，这将有助于提高flush的并行性。不过，我们认为目前的做法还不够好——尤其是LRU刷新上。在下一个我们介绍的Percona Server 5.7性能改进的文章中，我们将讲述MT（Multi-threaded）刷新方面的内容，以及为什么独立的MT LRU 刷新特别重要。
**`并行doublewrite buffer`**. 长期以来，MySQL只有一个doublewrite buffer来刷新数据页。所以，即使你有多个线程进行flush，也不能有效地使用他们——doublewrite会很快成为瓶颈。我们已经通过对每个buffer pool实例附加两个doublewrite buffer进行更改：一个对应一个类型的页面刷新（LRU和flush list）。这完全避免了任何doublewrite的争用，无论flush的线程数多少。我们还将doublewrite buffer移除系统表空间，以便现在可以配置其路径。

现在我们看下sysbench OLTP_RW，I/O绑定场景下的测试结果。下面是测试中使用的关键设置：
数据集 100 GB
innodb_buffer_pool_size=25GB
innodb_doublewrite=1
innodb_flush_log_at_trx_commit=1

![Sysbench OLTP_RW](https://www.percona.com/blog/wp-content/uploads/2016/03/5711.blog_.n1.v1.png)

当 [评估MySQL 5.7 RC](https://www.percona.com/blog/2015/10/26/state-percona-server-5-6-mysql-5-6-mysql-5-7rc/ )时，我们观察到I/O绑定场景中的性能下降，而且看起来类似MySQL 5.6的表现。性能下降的原因是buffer pool没有可用的空闲页。page clean 线程不能进行足够的LRU flush以便跟上需要，而且查询线程采用单页刷新的方式。这导致所有flush结构（尤其是doublewrite buffer）之间的
争用增加。

很长时间（Vadim十年前讨论过），InnoDB为大多数扩展性问题提供了统一的解决方案：`innodb_thread_concurrency` 系统变量。它允许你限制InnoDB内部活跃线程的数量并减少共享资源争用。随之而来，它也限制了最大可能达到的性能。

为了了解效果如何，我们用了两种不同的InnoDB并发设置进行两次测试：
**`innodb_thread_concurrency=0`**: 用这个默认值，Percona Server 5.7 表现最好，而MySQL 5.7 显示超过64并发之后性能就急剧下降。
**`innodb_thread_concurrency=64`**: 限制了InnoDB内部并发数，略微影响了Percona Server的吞吐量（不过与使用默认设置的想必，相差不大）。但是对于MySQL，这个改变是一个巨大的帮助。在64个并发之后，性能并没有下降，而是在达到4k并发时依然保持性能没很大下降（当然会有些波动）。

为了更好的了解细节，我们用512线程并发查看运行中的细节：

![Sysbench OLTP_RW,512线程](https://www.percona.com/blog/wp-content/uploads/2016/03/5711.blog_.n2.v4.png)

上图显示，争用会严重影响不设限制的并发吞吐量，但延迟的影响更加重要。限制并发有助于解决争用，但即使这样，Percona Server 还是有 15% ~ 25%的优势。

以下你可以看到上述每个运行的争用状况。这个图显示了每个同步对象（每秒）所有线程的累计等待时间。比如，所有图中，最热的对象是MySQL 5.7.11中的doublewrite互斥量（不带线程并发限制），每秒512线程大概有17s的等待时间。

![Top 5 performance schema synch waits](https://www.percona.com/blog/wp-content/uploads/2016/03/5711.blog_.n4.v6.png)

**mysql server 设置**
```
innodb_log_file_size=10G
innodb_doublewrite=1
innodb_flush_log_at_trx_commit=1
innodb_buffer_pool_instances=8
innodb_change_buffering=none
innodb_adaptive_hash_index=OFF
innodb_flush_method=O_DIRECT
innodb_flush_neighbors=0
innodb_read_io_threads=8
innodb_write_io_threads=8
innodb_lru_scan_depth=8192
innodb_io_capacity=15000
innodb_io_capacity_max=25000
loose-innodb-page-cleaners=4
table_open_cache_instances=64
table_open_cache=5000
loose-innodb-log_checksum-algorithm=crc32
loose-innodb-checksum-algorithm=strict_crc32
max_connections=50000
skip_name_resolve=ON
loose-performance_schema=ON
loose-performance-schema-instrument='wait/synch/%=ON',
```
###**总结**
如果你测试过MySQL 5.7，不妨再测试下Percona Server ——尤其在I/O绑定的工作负载下。我们在Percona Server 5.7的性能改进上煞费苦心。在后续发布的文章中，我们将深入了解LRU flush和doublewrite buffer变化的技术细节。

[原文英文链接](https://www.percona.com/blog/2016/03/17/percona-server-5-7-performance-improvements/)  
