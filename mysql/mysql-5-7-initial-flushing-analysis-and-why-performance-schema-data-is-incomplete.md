> 原文 [MySQL 5.7: initial flushing analysis and why Performance Schema data is incomplete](https://www.percona.com/blog/2016/05/03/mysql-5-7-initial-flushing-analysis-and-why-performance-schema-data-is-incomplete/)  
> 作者：Laurynas Biveinis and Alexey Stroganov   

In this post, we’ll examine why in an initial flushing analysis we find that Performance Schema data is incomplete.  
本文我们将阐述为什么在初始化的刷新分析中P_S数据不完整的原因。

[Having shown the performance impact of Percona Server 5.7 patches](https://www.percona.com/blog/2016/03/17/percona-server-5-7-performance-improvements/), we can now discuss their technical reasoning and details. Let’s revisit the MySQL 5.7.11 performance schema synch wait graph from the previous [post](https://www.percona.com/blog/2016/03/17/percona-server-5-7-performance-improvements/), for the case of unlimited InnoDB concurrency:  

[Percona Server 5.7 performance improvements](https://www.percona.com/blog/2016/03/17/percona-server-5-7-performance-improvements/) 文中已经表明了Percona Server 5.7补丁对于性能的影响，我们现在可以讨论它们的技术原理和细节。让我们从上文中回顾一下MySQL 5.7.11 performance schema synch wait曲线图，在这个测试中不限制InnoDB并发线程数：

![TOP 5 performance schema synch waits](https://www.percona.com/blog/wp-content/uploads/2016/03/5711.blog_.n6.v1.png)  

First of all, this graph is a little “nicer” than reality, which limits its diagnostic value. There are two reasons for this. The first one is that page cleaner worker threads are invisible to Performance Schema (see [bug 79894](http://bugs.mysql.com/bug.php?id=79894)). This alone limits PFS value in 5.7 if, for example, one tries to select only the events in the page cleaner threads or monitors low concurrency where the cleaner thread count is non-negligible part of the total threads.  

首先，这个曲线图看起来要比实际情况“好一些”，这使得它的诊断价值有限。这有两个原因。第一个是因为page cleaner线程在Performance Schema中不可见(见[bug 79894](http://bugs.mysql.com/bug.php?id=79894))。这个仅限于5.7中的PFS值，如果只查page cleanner线程中的事件，或者监视低并发性，其中cleaner线程数是整个线程的不可忽视的部分。

To understand the second reason, let’s look into PMP for the same setting. Note that selected intermediate stack frames were removed for clarity, especially in the InnoDB mutex implementation.  

为了理解第二个原因，让我们来看看相同设置的PMP。 请注意，为了清楚起见，移除了所选的中间堆栈帧，尤其是在InnoDB互斥实现中。

```
660 pthread_cond_wait,enter(ib0mutex.h:850),buf_dblwr_write_single_page(ib0mutex.h:850),buf_flush_write_block_low(buf0flu.cc:1096),buf_flush_page(buf0flu.cc:1096),buf_flush_single_page_from_LRU(buf0flu.cc:2217),buf_LRU_get_free_block(buf0lru.cc:1401),...
631 pthread_cond_wait,buf_dblwr_write_single_page(buf0dblwr.cc:1213),buf_flush_write_block_low(buf0flu.cc:1096),buf_flush_page(buf0flu.cc:1096),buf_flush_single_page_from_LRU(buf0flu.cc:2217),buf_LRU_get_free_block(buf0lru.cc:1401),...
337 pthread_cond_wait,PolicyMutex<TTASEventMutex<GenericPolicy>(ut0mutex.ic:89),get_next_redo_rseg(trx0trx.cc:1185),trx_assign_rseg_low(trx0trx.cc:1278),trx_set_rw_mode(trx0trx.cc:1278),lock_table(lock0lock.cc:4076),...
324 libaio::??(libaio.so.1),LinuxAIOHandler::collect(os0file.cc:2448),LinuxAIOHandler::poll(os0file.cc:2594),...
241 pthread_cond_wait,PolicyMutex<TTASEventMutex<GenericPolicy>(ut0mutex.ic:89),trx_write_serialisation_history(trx0trx.cc:1578),trx_commit_low(trx0trx.cc:2135),...
147 pthread_cond_wait,enter(ib0mutex.h:850),trx_undo_assign_undo(ib0mutex.h:850),trx_undo_report_row_operation(trx0rec.cc:1918),...
112 pthread_cod_wait,mtr_t::s_lock(sync0rw.ic:433),btr_cur_search_to_nth_level(btr0cur.cc:1008),...
83 poll(libc.so.6),Protocol_classic::get_command(protocol_classic.cc:965),do_command(sql_parse.cc:935),handle_connection(connection_handler_per_thread.cc:301),...
64 pthread_cond_wait,Per_thread_connection_handler::block_until_new_connection(thr_cond.h:136),...
```

The top wait in both PMP and the graph is the 660 samples of enter mutex inbuf_dblwr_write_single_pages, which is the doublewrite mutex. Now try to find the nearly as hot 631 samples of event wait inbuf_dblwr_write_single_page in the PFS output. You won’t find it because InnoDB OS event waits are not annotated in Performance Schema. In most cases this is correct, as OS event waits tend to be used when there is no work to do. The thread waits for work to appear, or for time to pass. But in the report above, the waiting thread is blocked from proceeding with useful work (see [bug 80979](http://bugs.mysql.com/bug.php?id=80979)).  

在PMP和图表中最多的等待事件是660个inbuf_dblwr_write_single_pages事件，这是doublewrite mutex。现在尝试在PFS中找到同样也很高的的631个inbuf_dblwr_write_single_page事件. 但却无法找到，因为InnoDB OS事件等待并不再PFS中记录。这在大多数情况下是没问题的，因为当InnoDB内部没事做时，则进入InnoDB OS wait事件状态。该线程要么等着出现工作时间，要么随着时间消逝。但是在上面的报告中，等待线程被阻止无法处理必要的工作(见[bug 80979](http://bugs.mysql.com/bug.php?id=80979))。

Now that we’ve shown the two reasons why PFS data is not telling the whole server story, let’s take PMP data instead and consider how to proceed. Those top two PMP waits suggest 1) the server is performing a lot of single page flushes, and 2) those single page flushes have their concurrency limited by the eight doublewrite single-page flush slots available, and that the wait for a free slot to appear is significant.

现在我们已经解释完了PFS数据中并没体现mysql全部状态的两个原因，那么让我们转而考虑PMP数据，并考虑如何进行。上面的两个PMP等待意味着，1)服务器执行大量的single page flush，2)这些single page flush的并发性受限于doublewrite中8个可用的single-page flush slot，并且很明显是在等待空闲的slot出现。

Two options become apparent at this point: either make the single-page flush doublewrite more parallel or reduce the single-page flushing in the first place. We’re big fans of the latter option since version 5.6 performance work, where we configured Percona Server to not perform single-page flushes at all by introducing the [innodb_empty_free_list_algorithm](https://www.percona.com/doc/percona-server/5.6/performance/xtradb_performance_improvements_for_io-bound_highly-concurrent_workloads.html) option, with the “backoff” default.  

这点上显然有两种选择：要么使single-page flush doublewrite更加并行，要么减少single-page flush。自5.6版本的性能优化开始，我们一直坚定后一种选择，在这里，我们通过引入 [innodb_empty_free_list_algorithm](https://www.percona.com/doc/percona-server/5.6/performance/xtradb_performance_improvements_for_io-bound_highly-concurrent_workloads.html)选项来配置Percona Server，使得不执行single-page flush，默认情况下是“backoff”。

The next post in the series will describe how we removed single-page flushing in 5.7.  
本系列的下一篇文章将描述如何删除5.7中的single-page flush。


 
     
