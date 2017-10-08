> 作者：Laurynas Biveinis and Alexey Stroganov    
> 发布时间：2016.5.5    
> 关键词：MySQL   

在这篇文章中，我们会讨论怎么利用多线程LRU刷新突破MySQL的瓶颈。

在[之前的文章](https://www.percona.com/blog/?p=34372)中，我们看到InnoDB 5.7执行大量的单页LRU刷新，它们又被共享的doublewrite buffer序列化。基于对5.6的经验判断，我们决定先挑战单页刷新这个问题。

首先我们先描述下单页刷新的概念。如果数据库实例的工作集大于可用的buffer pool，已经存在的数据页就要面临清理或者被刷（接着清理掉），从而为查询腾出空闲页。

InnoDB尝试通过维护每个buffer pool实例的空闲页列表来预测这一点。这些页面是可以立即用于放置新读取的数据页的。它的页面列表的长度由`innodb_lru_scan_depth`参数控制，并且清理线程通过执行LRU批量刷新来填充此列表。如果由于某种原因空闲页面的请求超出了清理线程的处理能力，server需要找到一个空闲的列表。试图不去阻止查询线程请求一个空闲页，它将执行单页的LRU刷新（buf_LRU_get_free_block在源码中调用buf_flush_single_page_from_LRU），这是在查询线程本身的上下文中执行的。

这种刷新模式的问题是，它会遍历buffer pools实例的LRU列表，同时在Innodb中持有buffer pool互斥量（或者XtraDB中更细粒度的LRU列表互斥体）。从而，当server的清理线程跟不上LRU刷新的需求就会进一步增加互斥量之前相互争用的压力——这也可以说是清理线程自己的问题。最后，一旦单页刷新找到一个页可以进行刷新，它在获取空闲的doublewrite buffer槽（如前所述）也还是会遇到问题。这就告诉我们一个道理，单页刷新并不是一个好的解决方案。下面的火焰图说明了一切：

![Flame Graph](https://www.percona.com/blog/wp-content/uploads/2016/03/512.io_.conc0_.svg)

这里注意，JOIN::optimize占据很大一块时间，其运行时间几乎完全由buf_dblwr_write_single_page在两个分支中完成。

最简单的避免单页刷新的方法是，额... 就是别去做它！等待清理线程最终提供一批空闲的页再提供给查询线程使用。这就是我们在XtraDB 5.6中添加了innodb_empty_free_list_algorithm选项（默认是"backoff"）。这个参数也存在于XtraDB 5.7，并解决了缓冲池（LRU列表）互斥量和doublewrite buffer单页刷新槽的争用问题。这种方法更好地处理了空的空闲页列表。

即使采用了这种策略，还有一个糟糕的情景是，当page clean线程无法跟上空闲页面的需求，它会导致查询阻塞。为了理解为什么会这样，让我们看下InnoDB 5.7中多线程LRU刷新的简单结构图：

![MySQL 5.7 MT LRU+flush list flushing](https://www.percona.com/blog/wp-content/uploads/2016/03/MySQL-MT-flushing-cropped.png)

从图中看，LRU的批量刷新不一定发生在最需要的时候。buffer pool实例首先刷新LRU列表（用于空闲页面），然后执行flush list刷新（用于checkpointh age和buffer pool脏页百分比维护）。如果flush list刷新正在执行，LRU刷新将不得不等到下一次刷新。此外，所有刷新都是有协调器线程每秒长时间迭代一次，等待一切完成。在error log中，如果一个工作线程停止了，可能一个1秒的标记会变成30秒或更多（比如提示："InnoDB: page_cleaner: 1000ms intended loop took 49812ms"）。所以如果我们有一个繁忙的buffer pool实例，所有都必须等待。而根据老司机的经验，缓冲池实例并不总是繁忙的（有时候忙，有时候闲）。

建议做如下修复：
* 将"LRU列表刷新"从"刷新列表刷新"中解耦，以便这两者在需要时可以并行执行。
* 认识到不同的缓冲池实例需要不同的刷新量，并删除实例之间的同步。

根据上述标准，我们为每个buffer pool实例分配一个独立的LRU刷新线程，该线程监视实例中的空闲列表长度，刷新，然后sleep等待下一个空闲列表长度检查。休眠时间根据空闲列表长度进行调整：因此，为了跟上需求，一个繁忙的buffer pool实例可能根本不会休眠，而比较闲的buffer pool实例刷新可能每秒钟唤醒一次，进行一次短暂的检查。

现在，LRU刷新结构如下:  
![Percona Server 5.7 MT LRU list flushing](https://www.percona.com/blog/wp-content/uploads/2016/03/Untitled-drawing-12.png)

这是在Percona Server 5.7.10-3 RC版本，这个设计也简化了代码。LRU的启发式刷新设计也是简洁的，现在任何的LRU刷新都从以前的清理coordinator/worker线程中移除——实现了更有效的flush list刷新。LRU刷新线程是可以刷新给定buffer pool的唯一线程，还可以进一步简化：比如，取消InnoDB 恢复写入线程。

那我们就大功告成了？No！随着单页刷新和单页刷新doublewrite瓶颈的消失，我们将再次转向doublewrite buffer。下篇文章见，拜拜！

[原文英文链接](https://www.percona.com/blog/2016/05/05/percona-server-5-7-multi-threaded-lru-flushing/)  
