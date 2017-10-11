>作者：Alexey Stroganov, Laurynas Biveinis and Vadim Tkachenko  
>发布时间：2016.3.17  
>文章关键字：Benchmarks,MySQL,Percona Server  
>原文：[Percona Server 5.7 performance improvements](https://www.percona.com/blog/2016/03/17/percona-server-5-7-performance-improvements/) 

In this blog post, we’ll be discussing Percona Server 5.7 performance improvements.  

在这篇文章中，我们将讨论Percona Server 5.7有哪些性能提升。

Starting from the Percona Server 5.6 release, we’ve introduced several significant changes that help address performance problems for highly-concurrent I/O-bound workloads. Some of our research and improvements were re-implemented for MySQL 5.7 – one of the best MySQL releases. But even though MySQL 5.7 showed progress in various aspects of scalability and performance, we’ve found that it’s possible to push I/O bound workload limits even further.  
  
从Percona Server 5.6发布以来，我们引入了几个重要的更新，有助于高并发I/O负载场景下的性能瓶颈定位。我们（在性能方面的）某些研究和提升在目前最好的MySQL版本5.7下被重新实现了。但即使MySQL 5.7在扩展性和性能等方便都有所提升，我们还是发现了可以增进I/O工作负载性能的一些地方。

Percona Server 5.7.11 currently has two major performance features in this area:  

Percona Server 5.7.11 有两个主要的性能方面的特性：

**Multi-threaded LRU flusher**. In a limited form, this feature exists in Percona Server 5.6. We split the LRU flusher thread out of the existing page cleaner thread, and it is now solely tasked with flushing the flush list. Along with several other important changes, this notably improved I/O bound workload performance. MySQL 5.7 has also made a step forward by introducing a pool of page cleaner threads that should help improve parallelism in flushing. However, we believe that the current approach is not good enough – especially for LRU flushing. In one of our next Percona Server 5.7 performance improvements posts, we’re going to describe aspects of MT flushing, and why it’s especially important to have an independent MT LRU flusher.  

**多线程LRU刷新**，这个特性在Percona Server 5.6就存在了，但效果有限。我们把LRU刷新线程从page cleaner线程中分离出来，现在完全只做flush list的刷新。除此外还有其他几个重要的变化，这些都显著提升了高I/O负载为主时的性能。MySQL 5.7更进一步引入了多个page cleaner线程机制，这将有助于提高flush的并行性。但我们还是认为目前的做法还不够好 —— 尤其是LRU list刷新方面。在我们下一篇介绍Percona Server 5.7性能改进的文章中，我们将讲述MT刷新（Multi-threaded）方面的内容，以及为什么独立的MT LRU 刷新特别重要。  

**Parallel doublewrite buffer**. For ages, MySQL has had only one doublewrite buffer for flushing data pages. So even if you had several threads for flushing you couldn’t efficiently use them – doublewrite quickly became a bottleneck. We’ve changed that by attaching two doublewrite buffers to each buffer pool instance: one for each type of page flushing (LRU and flush list). This completely avoids any doublewrite contention, regardless of the flusher thread count. We’ve also moved the doublewrite buffer out of the system tablespace so you can now configure its location.  

**并行doublewrite buffer**，长期以来，MySQL只有一个doublewrite buffer来刷新数据页。所以，即使你有多个线程进行flush，也不能有效地使用他们 —— doublewrite会很快成为瓶颈。我们则在每个buffer pool instance中实现两个doublewrite buffer，每个分别负责不同类型的页面刷新（LRU和flush list）。这完全避免了doublewrite的争用，无需考虑flush的线程数大小。我们还将doublewrite buffer从系统表空间中分离出来，可以自行配置其路径。

Now let’s review the results of a sysbench OLTP_RW, I/O-bound scenario. Below are the key settings that we used in our test:  

现在我们看下sysbench OLTP_RW，I/O负载为主场景下的测试结果。下面是测试中使用的关键设置:  

数据集大小：100GB  
```
innodb_buffer_pool_size=25GB  
innodb_doublewrite=1  
innodb_flush_log_at_trx_commit=1  
```

![Sysbench OLTP_RW](https://www.percona.com/blog/wp-content/uploads/2016/03/5711.blog_.n1.v1.png)

While evaluating MySQL 5.7 RC we observed a performance drop in I/O-bound workloads, and it looked very similar to MySQL 5.6 behavior. The reason for the drop is the lack of free pages in the buffer pool. Page cleaner threads are unable to perform enough LRU flushing to keep up with the demand, and the query threads resort to performing single page flushes. This results in increased contention between all the of the flushing structures (especially the doublewrite buffer).  

当 [评估MySQL 5.7 RC](https://www.percona.com/blog/2015/10/26/state-percona-server-5-6-mysql-5-6-mysql-5-7rc/ )时，我们观察到I/O负载为主场景中的性能下降，而且看起来类似MySQL 5.6的表现。性能下降的原因是buffer pool没有可用的空闲页。page cleaner 线程不能及时进行完成 LRU flush，而且查询线程又触发了单页刷新。这导致所有flushing structures（尤其是doublewrite buffer）之间的争用增加。

For ages (Vadim discussed this ten years ago!) InnoDB has had a universal workaround for most scalability issues: the innodb_thread_concurrency system variable. It allows you to limit the number of active threads within InnoDB and reduce shared resource contention. However, it comes with a trade-off in that the maximum possible performance is also limited.  

很长时间（Vadim十年前讨论过），InnoDB大多数扩展性问题都有统一的解决方案：`innodb_thread_concurrency` 选项。它可以限制InnoDB内部活跃线程的数量并减少共享资源争用。但是，but，它的缺陷也很明显，限制了可以发挥的最大性能。

To understand the effect, we ran the test two times with two different InnoDB concurrency settings:  

想了解其效果如何，我们在两种不同的InnoDB并发设置下进行对比测试：

**innodb_thread_concurrency=0**: with this default value Percona Server 5.7 shows the best results, while MySQL 5.7 shows sharply decreasing performance with more than 64 concurrent clients.  

**innodb_thread_concurrency=0**，这是默认值，Percona Server 5.7 表现最好，而MySQL 5.7 显示超过64并发之后性能就急剧下降。

**innodb_thread_concurrency=64**: limiting the number of threads inside InnoDB affects throughput for Percona Server slightly (with a small drop from the default setting), but for MySQL that setting change is a huge help. There were no drops in performance after 64 threads, and it’s able to maintain this performance level up to 4k threads (with some variance).  

**innodb_thread_concurrency=64**，限制了InnoDB内部并发数，略微影响Percona Server的吞吐量（和默认设置模式略有下降），但是对于MySQL这个改变有很大帮助。在64个并发之后，性能并没有下降，而是在达到4k并发时依然能保持这个性能表现（当然会有些波动）。

To understand the details better, let’s zoom into the test run with 512 threads:  

为了更好的了解细节，我们用512线程并发查看运行中的细节：

![Sysbench OLTP_RW,512线程](https://www.percona.com/blog/wp-content/uploads/2016/03/5711.blog_.n2.v4.png)

The charts above show that contentions significantly affect unrestricted concurrency throughput, but affect latency even worse. Limiting concurrency helps to address contentions, but even with this setting Percona Server shows 15-25% better.  

上图显示，争用会严重影响不设限制的并发吞吐量，但对响应延迟影响更大。限制并发有助于解决争用，但即使这样，Percona Server 还是有 15% ~ 25%的优势。

Below you can see the contention situation for each of the above runs. The graphs show total accumulated waiting time across all threads per synchronization object (per second). For example, the absolute hottest object across all graphs is the doublewrite mutex in MySQL-5.7.11 (without thread concurrency limitation). It has about 17 seconds of wait time across 512 client threads for each second of run time.  

以下你可以看到上述每次测试的争用情况。图中显示了所有线程每次同步对象（每秒）的累计等待时间。从所有图中可见，最热的对象是MySQL 5.7.11的doublewrite mutex（innodb_thread_concurrency=0），每秒并行512线程时共有大概有17s的等待时间。

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
### **总结**
If you are already testing 5.7, consider giving Percona Server a spin – especially if your workload is I/O bound. We’ve worked hard on Percona Server 5.7 performance improvements. In upcoming posts, we will delve into the technical details of our LRU flushing and doublewrite buffer changes.  

如果你测试过MySQL 5.7，不妨再测试下Percona Server ——尤其在I/O负载为主的场景。我们在Percona Server 5.7的性能改进上煞费苦心。在后续发布的文章中，我们将深入了解LRU flushing和doublewrite buffer上的一些变化等技术细节。 
