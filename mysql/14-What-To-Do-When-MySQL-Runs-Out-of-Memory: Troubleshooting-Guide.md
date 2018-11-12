>作者：Alexander Rubin  
>发布日期：2018-06-28  
>适用范围：MySQL, Percona Server for MySQL  
>关键词：memory, memory leaks, Memory Usage, MySQL server memory usage, MySQL Troubleshooting, Troubleshooting MySQL, troubleshooting tips  


Troubleshooting crashes is never a fun task, especially if MySQL does not report the cause of the crash. For example, when MySQL runs out of memory. Peter Zaitsev wrote a blog post in 2012: Troubleshooting MySQL Memory Usage with a lots of useful tips. With the new versions of MySQL (5.7+) and performance_schema we have the ability to troubleshoot MySQL memory allocation much more easily。  
崩溃故障诊断绝不是一个有趣的任务,尤其如果MySQL没有报告崩溃的原因时,比如,MySQL运行时内存溢出。 Peter Zaitsev 在2012年写了一篇文章[Troubleshooting MySQL Memory Usage ](https://www.percona.com/blog/2012/03/21/troubleshooting-mysql-memory-usage/)里面有很多有用的提示.使用新版本MySQL(5.7+)结合performance_schema,我们可以更轻松地解决MySQL内存分配问题。  

In this blog post I will show you how to use it.  

First of all, there are 3 major cases when MySQL will crash due to running out of memory:  

1. MySQL tries to allocate more memory than available because we specifically told it to do so. For example: you did not set innodb_buffer_pool_size correctly. This is very easy to fix  
2. There is some other process(es) on the server that allocates RAM. It can be the application (java, python, php), web server or even the backup (i.e. mysqldump). When the source of the problem is identified, it is straightforward to fix.  
3. Memory leaks in MySQL. This is a worst case scenario, and we need to troubleshoot.  

在这篇博文中,我将向你展示如何使用它。  

首先,MySQL因为内存溢出发生崩溃主要有以下三种情况：  

1. MySQL试图分配比可用内存更多的内存,因为我们特意告诉它这样做.比如你没有正确的设置innodb_buffer_pool_size.这种情况很好解决。  
2. 服务器上有其他一些进程分配了RAM内存.可能是应用程序(java,python,php),web服务器,或者甚至备份(比如mysqldump).确定问题的根源
后,可以直接修复。  
3. MySQL内存泄漏.这时最糟糕的情况,这时需要我们进行故障诊断。  

### Where to start troubleshooting MySQL memory leaks  
### 从哪里开始诊断MySQL内存泄漏的问题  

Here is what we can start with (assuming it is a Linux server):  
假设是一个linux服务器,我们可以从以下开始:  

**Part 1: Linux OS and config check**  
1. Identify the crash by checking mysql error log and Linux log file (i.e. /var/log/messages or /var/log/syslog). You may see an entry saying that OOM Killer killed MySQL. Whenever MySQL has been killed by OOM “dmesg” also shows details about the circumstances surrounding it.  
2. Check the available RAM:  
	* free -g  
	* cat /proc/meminfo  
3. Check what applications are using RAM: “top” or “htop” (see the resident vs virtual memory)  
4. Check mysql configuration: check /etc/my.cnf or in general /etc/my* (including /etc/mysql/* and other files). MySQL may be running with the different my.cnf (run ps  ax| grep mysql )  
5. Run vmstat 5 5 to see if the system is reading/writing via virtual memory and if it is swapping  
6. For non-production environments we can use other tools (like Valgrind, gdb, etc) to examine MySQL usage  

**第一部分: Linux 系统和配置检查**  

1. 通过检查mysql error日志和linux日志(比如,/var/log/messages 或者 /var/log/syslog)确认崩溃.  
你可能会看到一条条目说OOM Killer杀死了MySQL.每当MySQL被OOM杀死时，“dmesg”也会显示有关它周围情况的详细信息  
2. 检查可用的RAM内存:  
	* free -g  
	* cat /proc/meminfo  
3. 检查什么程序在使用内存:"top"或者htop(看resident和virtual列)  
4. 检查mysql的配置:检查/etc/my.cnf或者一般的/etc/my*(包括/etc/mysql/*和其他文件).  
MySQL 可能跟着不同的my.cnf运行（用ps ax | grep mysql)  
5. 运行vmstat 5 5 查看系统是否通过虚拟内存进行读写以及是否正在进行swap交换  
6. 对于非生产环境,我们可以使用其他工具(如Valgrind,gdb等)来检查MySQL的使用情况.  

**Part 2:  Checks inside MySQL**  
Now we can check things inside MySQL to look for potential MySQL memory leaks.  
MySQL allocates memory in tons of places. Especially:  
* Table cache  
* Performance_schema (run: show engine performance_schema status  and look at the last line). That may be the cause for the systems with small amount of RAM, i.e. 1G or less  
* InnoDB (run show engine innodb status  and check the buffer pool section, memory allocated for buffer_pool and related caches)  
* Temporary tables in RAM (find all in-memory tables by running: select * from information_schema.tables where engine='MEMORY' )  
* Prepared statements, when it is not deallocated (check the number of prepared commands via deallocate command by running show global status like ‘ Com_prepare_sql';show global status like 'Com_dealloc_sql'  )  

**第二部分: 检查MySQL内部** 

现在我们可以检查MySQL内部的东西来寻找潜在的MySQL内存泄漏情况:  
MySQL在很多地方分配内存.尤其：  
 * 表缓存  
 * Performance_schema(运行:show engine performance_schema status 然后看最后一行).这可能在系统RAM比较少(1G或更少)时的可能原因.  
 * InnoDB(运行show engine innodb status 检查 buffer pool部分,为buffer pool及相关缓存分配的内存)  
 * 内存中的临时表(查看所有内存表:select * from information_schema.tables where engine='MEMORY')  
 * 预处理语句,当他们没有被释放时(通过运行show global status like 'Com_prepare_sql'和show global status like 'Com_dealloc_sql'来检查通过deallocate命令释放的预处理语句)  

The good news is: starting with MySQL 5.7 we have memory allocation in performance_schema. Here is how we can use it.  
好消息是,从5.7开始我们可以通过performance_schema查看内存的分配情况.下面就展示如何使用它.  

1. First, we need to enable collecting memory metrics. Run:  

```
UPDATE setup_instruments SET ENABLED = 'YES' WHERE NAME LIKE 'memory/%';  
```

2. Run the report from sys schema:  

```
select event_name, current_alloc, high_alloc from sys.memory_global_by_current_bytes where current_count > 0;  
```  

3. Usually this will give you the place in code when memory is allocated. It is usually self-explanatory. In some cases we can search for bugs or we might need to check the MySQL source code.  
---

1. 首先,我们需要启用收集内存指标,运行如下语句:  

```
UPDATE setup_instruments SET ENABLED = 'YES' WHERE NAME LIKE 'memory/%';  
```

2. 运行sys schema里面的报告  

```
select event_name,current_alloc,high_alloc from sys.memory_global_by_current_bytes where current_count > 0;  
```

3. 通常,这将在分配内存时为你提供代码,它通常是不言自明的.在某些情况下,我们可以搜索错误,或者我们可能需要检查MySQL源代码.  

For example, for the bug where memory was over-allocated in triggers ([https://bugs.mysql.com/bug.php?id=86821](https://bugs.mysql.com/bug.php?id=86821)) the select shows:  
例如,有一个过度为触发器分配内存的bug([https://bugs.mysql.com/bug.php?id=86821](https://bugs.mysql.com/bug.php?id=86821))  

```
mysql> select event_name, current_alloc, high_alloc from memory_global_by_current_bytes where current_count > 0;  
+--------------------------------------------------------------------------------+---------------+-------------+
| event_name                                                                     | current_alloc | high_alloc  |
+--------------------------------------------------------------------------------+---------------+-------------+
| memory/innodb/buf_buf_pool                                                     | 7.29 GiB      | 7.29 GiB    |
| memory/sql/sp_head::main_mem_root                                              | 3.21 GiB      | 3.62 GiB    |
...
```

查询的显示如下:  

```
mysql> select event_name, current_alloc, high_alloc from memory_global_by_current_bytes where current_count > 0;
+--------------------------------------------------------------------------------+---------------+-------------+
| event_name                                                                     | current_alloc | high_alloc  |
+--------------------------------------------------------------------------------+---------------+-------------+
| memory/innodb/buf_buf_pool                                                     | 7.29 GiB      | 7.29 GiB    |
| memory/sql/sp_head::main_mem_root                                              | 3.21 GiB      | 3.62 GiB    |
...
```

The largest chunk of RAM is usually the buffer pool but ~3G in stored procedures seems to be too high.  
分配最大一块内存通常是buffer pool，但是约3G的存储过程似乎有点太高了.  

According to the [MySQL source code documentation](https://dev.mysql.com/doc/dev/mysql-server/8.0.0/classsp__head.html#details), sp_head represents one instance of a stored program which might be of any type (stored procedure, function, trigger, event). In the above case we have a potential memory leak.  
根据[MySQL source code documentation](https://dev.mysql.com/doc/dev/mysql-server/8.0.0/classsp__head.html#details),sp_head表示存储程序里面的一个实例(比如存储过程,函数,触发器,事件).在上面的例子,我们有潜在的内存泄漏的风险.  

In addition we can get a total report for each higher level event if we want to see from the birds eye what is eating memory:  
另外，我们想要鸟瞰什么吃掉了内存，我们可以获得每个事件更高级别活动的总体报告.  

```
mysql> select  substring_index(
    ->     substring_index(event_name, '/', 2),
    ->     '/',
    ->     -1
    ->   )  as event_type,
    ->   round(sum(CURRENT_NUMBER_OF_BYTES_USED)/1024/1024, 2) as MB_CURRENTLY_USED
    -> from performance_schema.memory_summary_global_by_event_name
    -> group by event_type
    -> having MB_CURRENTLY_USED>0;
+--------------------+-------------------+
| event_type         | MB_CURRENTLY_USED |
+--------------------+-------------------+
| innodb             |              0.61 |
| memory             |              0.21 |
| performance_schema |            106.26 |
| sql                |              0.79 |
+--------------------+-------------------+
4 rows in set (0.00 sec)
```

I hope those simple steps can help troubleshoot MySQL crashes due to running out of memory.  
我希望这些简单的步骤可以帮助解决由于内存溢出导致的MySQL崩溃问题。  
