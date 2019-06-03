# MySQL内存管理，内存分配器和操作系统
> 原文 ：[MySQL Memory Management, Memory Allocators and Operating System](<https://www.percona.com/blog/2019/05/02/mysql-memory-management-memory-allocators-and-operating-system/>)
>
> 作者：[Sveta Smirnova](https://www.percona.com/blog/author/sveta-smirnova/)
>
> 翻译：郑志江
>
> 校对：徐晨亮

*When users experience memory usage issues with any software, including MySQL®, their first response is to think that it’s a symptom of a memory leak. As this story will show, this is not always the case.*

*This story is about a bug*

当用户使用软件(包括MySQL)碰到内存问题时，我们第一反应就是内存泄漏。正如这篇文章所示，并不总是这样。

这篇文章阐述一个关于内存的bug。

*All Percona Support customers are eligible for bug fixes, but their options vary. For example, [Advanced+](https://www.percona.com/services/support/support-tiers-mysql) customers are offered a HotFix build prior to the public release of software with the patch. [Premium](https://www.percona.com/services/support/support-tiers-mysql) customers do not even have to use Percona software: we may port our patches to upstream for them. But for Percona products all Support levels have the right to have a fix.*

所有的percona支持的客户都有获得bug修复的资格，但他们也有不同的选择。比如，超级客户在软件补丁正式发布之前就可以获得hotfiix版本，优质客户甚至不需要使用percona的软件，我们也可以为他们把补丁推到上游。但对于与percona产品来说，所有支持等级都有权得到bug修复。

*Even so, this does not mean we will fix every unexpected behavior, even if we accept that behavior to be a valid bug. One of the reasons for such a decision might be that while the behavior is clearly wrong for Percona products, this is still a feature request.*

即便如此，这并不意味着我们会修复所有的意外情况，即使我们接受这种情况为一个有效bug。做出这样的决定的原因之一可能是这个意外情况虽然很明确是错误的，但对于percona产品本身来说确实一个产品需求

## 作为学习案例的一个bug

*A good recent example of such a case is [PS-5312](https://jira.percona.com/browse/PS-5312) – the bug is repeatable with upstream and reported at [bugs.mysql.com/95065*](https://bugs.mysql.com/bug.php?id=95065)

最近一个很好的案例是 [PS-5312](https://jira.percona.com/browse/PS-5312)——这个bug可在上游复现并被记录在[bugs.mysql.com/95065](https://www.percona.com/blog/2019/05/02/mysql-memory-management-memory-allocators-and-operating-system/)。

*This reports a situation whereby access to InnoDB fulltext indexes leads to growth in memory usage. It starts when someone queries a fulltext index, grows until a maximum, and is not freed for quite a long time.*

这个报告阐述了一种情况，当访问InnoDB的全文索引的时候会导致内存使用量增长。这种情况出现在一些全文索引的查询，内存会持续增长直到达到最大值，并且很长时间不会释放。

[*Yura Sorokin](https://jira.percona.com/secure/ViewProfile.jspa?name=yura.sorokin) from the Percona Engineering Team investigated if this is a memory leak and found that it is not.*

来自Percona工程团队的Yura Sorokin研究表明，这种情况并不属于内存泄漏范畴。

*When InnoDB resolves a fulltext query, it creates a memory heap in the function fts_query_phrase_search This heap may grow up to 80MB. Additionally, it has a big number of blocks ( mem_block_t ) which are not always used continuously and this, in turn, leads to memory fragmentation.*

当InnoDB解析一个全文查询时，它会创建在`fts_query_phrase_search`函数中创建一个内存堆，这个堆可能增长到80M。另外，这个过程还会使用到大量非连续块(`mem_block_t`)进而产生的内存碎片。

*In the function exit , the memory heap is freed. InnoDB does this for each of the allocated blocks. At the end of the function, it calls free() which belongs to one of the memory allocator libraries, such as malloc or [jemalloc](http://jemalloc.net/). From the mysqld point of view, everything is done correctly: there is no memory leak.*

在函数出口，这些内存堆会被释放。InnoDB会为其分配的每一个块做这个操作。在函数结束时，会调用一个内存分配器库中的`free()`操作，比如`malloc`或者`jemalloc`。从MySQL本身来看，这都是没问题的，不存在内存泄漏。

*However while free() should release memory when called, it is not required to return it back to the operating system. If the memory allocator decides that the same memory blocks will be required soon, it may still keep them for the mysqld process. This explains why you might see that mysqld  still uses a lot of memory after the job is finished and all de-allocations are done.*

然而，调用`free()`确实时是应该释放内存，但不需要将其返回给操作系统。如果内存分配器发现这些内存块马上还需要被用到，则会将他们保留住继续用于mysqld进程。这就解释了为什么mysqld在job完成且所有取消资源分配都结束后还会占用大量内存。

*This in practice is not a big issue and should not cause any harm. But if you need the memory to be returned to the operating system quicker, you could try alternative memory allocators, such as [jemalloc](http://jemalloc.net/). The latter was proven to solve the issue with [PS-5312](https://jira.percona.com/browse/PS-5312).*

这个在实际生产中并不是一个大问题，按道理不应该造成任何事故。但是如果你需要更快地将内存返回给操作系统，你可以尝试非传统的内存分配器，类似`jemallolc`。它被证明可以解决[PS-5312](https://jira.percona.com/browse/PS-5312)的问题。

*Another factor which improves memory management is the number of CPU cores: the more we used for the test, the faster the memory was returned to the operating system. This, probably, can be explained by the fact that if you have multiple CPUs, then the memory allocator can dedicate one of them just for releasing memory to the operating system.*

另一个改善内存管理的因素是cpu内核数量：在测试中，cpu核数越多，内存返回给操作系统的速度会越快。这可能是你拥有多个CPU，则其中一个可专门用作内存分配器释放内存到操作系统。

*The very first implementation of InnoDB full text indexes introduced this flaw. As our engineer Yura Sorokin [found](https://jira.percona.com/browse/PS-5312?focusedCommentId=236644&page=com.atlassian.jira.plugin.system.issuetabpanels:comment-tabpanel#comment-236644):*

> - The very first 5.6 commit which introduces Full Text Search Functionality for InnoDB WL#5538: InnoDB Full-Text Search Support – <https://dev.mysql.com/worklog/task/?id=5538>
> - Implement WL #5538 InnoDB Full-Text Search Support, merge – [https://github.com/mysql/mysql-server/commit/b6169e2d944 ](https://github.com/mysql/mysql-server/commit/b6169e2d944)– also has this problem.

正如我们的工程师`Yura Sorokin`所发现的一样，下面两点阐述了InnoDB全文索引的早期实现引入了这个缺陷：

- 5.6版本MySQL最早对InnoDB WL全文索引功能引入的介绍：#5538: InnoDB全文搜索支持 – https://dev.mysql.com/worklog/task/?id=5538
- 实现WL #5538 InnoDB全文搜索支持与合并 - <https://github.com/mysql/mysql-server/commit/b6169e2d944> - 也存在同样的问题问题

## 修复方法

*We have a few options to fix this:*

1. *Change implementation of InnoDB fulltext index*
2. *Use custom memory library like jemalloc*

*Both have their advantages and disadvantages.*

我们有两种方法来修复这个问题：

​	1.修改InnoDB全文索引的实现

​	2.使用自定义内存库，例如`jemalloc`

这两种方法都有各自的优缺点。

**Option 1** *means we are introducing an incompatibility with upstream, which may lead to strange bugs in future versions. This also means a full rewrite of the InnoDB fulltext code which is always risky in GA versions, used by our customers.*

**方法1** 意味着我们引入了与软件上游不兼容性的风险，这可能会导致新版本中出现未知的错误。也意味着彻底重写InnoDB全文索引部分代码，这在用户们使用的GA版本中是有风险的。

**Option 2** *means we may hit flaws in the [jemalloc](https://jira.percona.com/browse/PS-5312) library which is designed for performance and not for the safest memory allocation.*

**方法2** 则也为着我们可能会命中一些jemalloc库中性能大于安全的策略缺陷。

*So we have to choose between these two not ideal solutions.*

*Since **option 1** may lead to a situation when Percona Server will be incompatible with upstream, we prefer **option 2**and look forward for the upstream fix of this bug.*

因此我们不得不在这两个并不完美的方法中选择一个。

鉴于方法一可能导致percona服务与上游的不兼容，我们更倾向于用方法二来解决问题，并期待着上游修复这个bug。

## 结论

*If you are seeing a high memory usage by the mysqld process, it is not always a symptom of a memory leak. You can use memory instrumentation in Performance Schema to find out how allocated memory is used. Try alternative memory libraries for better processing of allocations and freeing of memory. Search the user manual for LD_PRELOADto find out how to set it up at these pages [here](https://dev.mysql.com/doc/refman/8.0/en/mysqld-safe.html) and [here](https://dev.mysql.com/doc/mysql-installation-excerpt/8.0/en/using-systemd.html).*

如果发现mysqld进程占用内存很高，并不代表一定是内存泄漏。我们可以在Performance Schema中使用内存检测来了解今进程是如何使用已分配的内存。也可以尝试替换内存库来更好地处理内存分配与释放。关于LD_RELOAD如何配置，请查阅MySQL用户手册对应页面 [mysqld-safe](https://dev.mysql.com/doc/refman/8.0/en/mysqld-safe.html)和[using-system](https://dev.mysql.com/doc/mysql-installation-excerpt/8.0/en/using-systemd.html)。









