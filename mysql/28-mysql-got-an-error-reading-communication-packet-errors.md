# MySQL “Got an error reading communication packet”

> 作者：[Muhammad Irfan](https://www.percona.com/blog/author/mirfan/)
>
> 原文地址：https://www.percona.com/blog/2016/05/16/mysql-got-an-error-reading-communication-packet-errors/
>
> 翻译：徐晨亮



In this blog post, we’ll discuss the possible reasons for MySQL “Got an error reading communication packet” errors and how to address them.

在这篇文章中，我们将会讨论MySQL"Got an error reading communication packet"错误的原因以及如何定位它们。

In Percona’s managed services, we often receive customer questions on communication failure errors. So let’s discuss possible reasons for this error and how to remedy it.

在Percona的托管服务中，我们经常收到关于通信失败错误的客户咨询。因此让我们一起来讨论下该错误可能的原因以及如何来规避。

##MySQL Communication Errors

First of all, whenever a communication error occurs, it increments the status counter for either [Aborted_clients](http://dev.mysql.com/doc/refman/5.6/en/server-status-variables.html#statvar_Aborted_clients) or [Aborted_connects](http://dev.mysql.com/doc/refman/5.6/en/server-status-variables.html#statvar_Aborted_connects), which describe the number of connections that were aborted because the client died without closing the connection properly and the number of failed attempts to connect to MySQL server (respectively). The possible reasons for both errors are numerous (see the Aborted_clients increments or Aborted_connects increments sections in the MySQL [manual](http://dev.mysql.com/doc/refman/5.6/en/communication-errors.html)).

首先，不论何时通信错误发生，状态计数器`Aborted clients`或者`Aborted connects`都会增加，表示由于客户端未正确关闭连接而断开的次数，以及连接到MySQL服务器失败的尝试次数。这两个错误的可能原因有很多(参见MySQL [手册](http://dev.mysql.com/doc/refman/5.6/en/communication-errors.html)中的Aborted_clients increments或aborted_connections increments部分）

In the case of [log_warnings](http://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html#sysvar_log_warnings), MySQL also writes this information to the error log (shown below):

在[log_warnings](http://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html#sysvar_log_warnings)>1的情况下，MySQL同样将该信息写入到error log（如下所示）:

```mysql
[Warning] Aborted connection 305628 to db: 'db' user: 'dbuser' host: 'hostname' (Got an error reading communication packets)
[Warning] Aborted connection 305627 to db: 'db' user: 'dbuser' host: 'hostname' (Got an error reading c
```

In this case, MySQL increments the status counter for Aborted_clients, which could mean:

- The client connected successfully but terminated improperly (and may relate to not closing the connection properly)
- The client slept for longer than the defined [wait_timeout](http://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html#sysvar_wait_timeout) or [interactive_timeout](http://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html#sysvar_interactive_timeout) seconds (which ends up causing the connection to sleep for wait_timeout seconds and then the connection gets forcibly closed by the MySQL server)
- The client terminated abnormally or exceeded the [max_allowed_packet](https://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html#sysvar_max_allowed_packet) for queries

The above is not an all-inclusive list. Now, let’s identify what is causing this problem and how to remedy it.



以下情况下， MySQL会增加Aborted_clients状态变量的计数器，也就意味着：

- 客户端已经成功连接，但是异常终止了（可能与未正确关闭连接有关系）

-  客户端sleep时间超过了变量[wait_timeout](http://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html#sysvar_wait_timeout)或 [interactive_timeout](http://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html#sysvar_interactive_timeout)定义的秒数（最终导致连接休眠的时间超过系统变量wait_timeout的值，然后被MySQL强行关闭）
- 客户端异常中断或查询超出了 [max_allowed_packet](https://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html#sysvar_max_allowed_packet)值

以上是一个非全部包含的原因列表。现在让我们找出造成该问题的原因并且如何规避。



## Fixing MySQL Communication Errors

To be honest, aborted connection errors are not easy to diagnose. But in my experience, it’s related to network/firewall issues most of the time. We usually investigate those issues with the help of Percona toolkit scripts, i.e. [pt-summary](https://www.percona.com/doc/percona-toolkit/2.2/pt-summary.html) / [pt-mysql-summary](https://www.percona.com/doc/percona-toolkit/2.2/pt-mysql-summary.html) / [pt-stalk](https://www.percona.com/doc/percona-toolkit/2.2/pt-stalk.html). The outputs from those scripts can be very helpful.

说实话，aborted connections错误并不容易诊断。但是在我的经验中，大多数时间是网络或者防火墙的原因。我们通常使用Percona toolkit脚本，例如 [pt-summary](https://www.percona.com/doc/percona-toolkit/2.2/pt-summary.html) / [pt-mysql-summary](https://www.percona.com/doc/percona-toolkit/2.2/pt-mysql-summary.html) / [pt-stalk](https://www.percona.com/doc/percona-toolkit/2.2/pt-stalk.html)来调查这些问题。这些脚本的输出非常有用。

Some of the reasons for aborted connection errors can be:

- A high rate of connections sleeping inside MySQL for hundred of seconds is one of the symptoms that applications aren’t closing connections after doing work, and instead relying on the wait_timeout to close them. I strongly recommend changing the application logic to properly close connections at the end of an operation.

- Check to make sure the value of max_allowed_packet is high enough, and that your clients are not receiving a “packet too large” message. This situation aborts the connection without properly closing it.

- Another possibility is TIME_WAIT. I’ve noticed many TIME_WAIT notifications from the netstat, so I would recommend confirming the connections are well managed to close on the application side.

- Make sure the transactions are committed (begin and commit) properly so that once the application is “done” with the connection it is left in a clean state.

- You should ensure that client applications do not abort connections. For example, if PHP has option max_execution_time set to 5 seconds, increasing [connect_timeout](https://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html#sysvar_connect_timeout) would not help because PHP will kill the script. Other programming languages and environments can have similar safety options.

- Another cause for delay in connections is DNS problems. Check if you have [skip-name-resolve](http://dev.mysql.com/doc/refman/5.6/en/server-options.html#option_mysqld_skip-name-resolve) enabled and if hosts are authenticated against their IP address instead of their hostname.

- One way to find out where your application is misbehaving is to add some logging to your code that will save the application actions along with the MySQL connection ID. With that, you can correlate it to the connection number from the error lines. Enable the Audit log plugin, which logs connections and query activity, and check the [Percona Audit Log Plugin](https://www.percona.com/doc/percona-server/5.6/management/audit_log_plugin.html) as soon as you hit a connection abort error. You can check for the audit log to identify which query is the culprit. If you can’t use the Audit plugin for some reason, you can consider using the MySQL general log – however, this can be risky on a loaded server. You should enable the [general log](http://dev.mysql.com/doc/refman/5.6/en/query-log.html) for at least a few minutes. While it puts a heavy burden on the server, errors happen fairly often so you should be able to collect the data before the log grows too large. I recommend enabling the general log with an -f tail, then disable the general log when you see the next warning in the log. Once you find the query from the aborted connection, identify which piece of your application issues that query and co-relate the queries with portions of your application.

- Try increasing the [net_read_timeout](https://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html#sysvar_net_read_timeout) and [net_write_timeout](https://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html#sysvar_net_write_timeout) values for MySQL and see if that reduces the number of errors. net_read_timeout is rarely the problem unless you have an extremely poor network. Try tweaking those values, however, because in most cases a query is generated and sent as a single packet to the server, and applications can’t switch to doing something else while leaving the server with a partially received query. There is a very detailed [blog post](https://www.percona.com/blog/2007/07/08/mysql-net_write_timeout-vs-wait_timeout-and-protocol-notes/) on this topic from our CEO, Peter Zaitsev.

  

以下是造成aborted connection错误的可能原因：

- 在MySQL内部，MySQL内部处于休眠了几百秒的状态的连接中很大比例是应用程序在做完工作后没有关闭连接造成的，而是依靠wait_tiemout系统变量来关闭连接。 我强烈建议修改应用程序逻辑，在操作结束后正确关闭连接
- 检查以确保`max_allowed_packet`足够大，你的客户端不会收到"packet too large"的消息。这种情况下的连接断开属于由于没有正确关闭连接
- 另外一种可能是`TIME_WAIT`。我曾经多次从netstat注意到`TIME_WAIT`提示，所以我建议在应用端确认很好地管理来关闭连接
- 确保事务提交（begin和commit）都正确提交以保证一旦应用程序完成以后留下的连接是处于干净的状态
- 你应该确保客户端程序不会断开连接。例如，如果PHP设置了`max_execution_time`为5秒，增加`connect_timeout`并不会起到作用，因为PHP会kill脚本。其他程序语言和环境也有类似的安全选项
- 连接延迟的另外一个原因是DNS问题。检查参数`skip-name-resolve`是否打开，以及是否根据主机的IP地址而不是主机名对主机进行身份验证
- 发现你的应用程序故障的一种办法是添加一些日志到你的代码中来保存包含连接ID的应用程序行为。有了它，你能够将连接数字与错误行数对应起来了。打开审计日志插件，记录了连接和查询操作，一旦触发到了连接断开的错误，你应该检查Percona审计日志。你可以通过检查审计日志找出哪个查询是根本原因。如果由于某些原因你不能使用审计日志，你可以考虑使用MySQL的常规日志-然而对于高负载的服务器来说这样是有风险的。再不济，你可以打开常规日志几分钟。打开常规日志会给服务器增加巨大负担，并且经常会发生错误，因此你应该在日志增长太大之前就收集完数据。我建议来打开常规日志并使用-f tail，然后当你在日志中看到下一个警告时关闭。一旦从断开的连接中找到查询，请确定查询的应用程序问题的哪一部分，并将查询与应用程序的某些部分关联起来。
- 尝试增加MySQL的`net_read_timeout`和`net_write_timeout`的参数值然后观察是否减少错误数。`net_read_timeout`一般很少出问题，除非你的网络真的很糟糕。但是，尝试调整这些值，因为在大多数情况下，生成一个查询并将其作为一个包发送到服务器，而应用程序不能在将部分接收到的查询留给服务器的同时去做其他事情



Aborted connections happen because a connection was not closed properly. The server can’t cause aborted connections unless there is a networking problem between the server and the client (like the server is half duplex, and the client is full duplex) – but that is the network causing the problem, not the server. In any case, such problems should show up as errors on the networking interface. To be extra sure, check the  ifconfig -a  output on the MySQL server to check if there are errors.

发生连接断开的原因是因为连接没有正确关闭。服务器并不能造成连接断开，除非服务器和客户端之间有网络问题（例如服务器是单工而客户端是双工的）-但是这是网络造成的问题，而不是服务器。在任何情况下，这些问题都应该在网络接口上显示为错误。另外，请检查MySQL服务器上的`ifconfig -a`输出是否有错误。

Another way to troubleshoot this problem is via tcpdump. You can refer to this blog post on [how to track down the source of aborted connections](https://www.percona.com/blog/2008/08/23/how-to-track-down-the-source-of-aborted_connects/). Look for potential network issues, timeouts and resource issues with MySQL.

另外一种定位该问题的方法是通过tcpdump。你可以参考这篇文章[how to track down the source of aborted connections](https://www.percona.com/blog/2008/08/23/how-to-track-down-the-source-of-aborted_connects/)找到MySQL的潜在网络、超时和资源问题。

I found this [blog post](https://www.percona.com/blog/2011/04/18/how-to-use-tcpdump-on-very-busy-hosts/) useful in explaining how to use tcpdump on busy hosts. It provides help for tracking down the TCP exchange sequence that led to the aborted connection, which can help you figure out why the connection broke.

我发现了 [这篇](https://www.percona.com/blog/2011/04/18/how-to-use-tcpdump-on-very-busy-hosts/)关于解释如何在一台繁忙的机器上使用tcpdump的文章非常有用。它提供了跟踪导致断开连接的TCP交换序列的帮助，能够帮你找出连接中断的原因。

For network issues, use a ping to calculate the round trip time (RTT) between a machine where mysqld is located and the machine from where the application makes requests. Send a large file (1GB or more) to and from client and server machines, watch the process using tcpdump, then check if an error occurred during transfer. Repeat this test a few times. I also found this from my colleague Marco Tusa useful: [Effective way to check network connection](http://www.tusacentral.net/joomla/index.php/mysql-blogs/164-effective-way-to-check-the-network-connection-when-in-need-of-a-geographic-distribution-replication-.html).

对于网络问题，使用`ping`来计算从发起请求的应用服务器到mysqld服务器间的往返时间(RTT)。从客户端发送一个大文件（1GB或者更大）到服务端，使用tcpdump观察进程，并检查传输期间是否有错误发生。重复测试数次。我还发现我同事Marco Tusa的文章也非常有用[Effective way to check network connection](http://www.tusacentral.net/joomla/index.php/mysql-blogs/164-effective-way-to-check-the-network-connection-when-in-need-of-a-geographic-distribution-replication-.html)

One other idea I can think of is to capture the  netstat -s output along with a timestamp after every N seconds (e.g., 10 seconds so you can relate  netstat -s output of BEFORE and AFTER an aborted connection error from the MySQL error log). With the aborted connection error timestamp, you can co-relate it with the  netstat sample captured as per a timestamp of netstat, and watch which error counters increased under the TcpExt section of netstat -s.

我能想到的另外一种思路是每隔N秒抓取`netstat -s`加上时间戳的输出（例如隔10秒，你可以将`netstat -s`的输出与MySQL的错误日志中的连接断开错误前后联系起来）。通过断开连接错误时间戳，你可以将它与捕捉到的带时间戳的netstat示例关联起来，并观察在netstat -s的TcpExt部分中哪些错误计数器增加了。

Along with that, you should also check the network infrastructure sitting between the client and the server for proxies, load balancers, and firewalls that could be causing a problem.

与此同时，你还应该检查客户机和服务器之间的网络基础设施，以查找可能导致问题的代理、负载均衡和防火墙。

**Conclusion:**In addition to diagnosing communication failure errors, you also need to take into account faulty ethernets, hubs, switches, cables, and so forth which can cause this issue as well. You must replace the hardware itself to properly diagnose these issues.

**结论：** 除了诊断通信故障错误之外，还需要考虑网卡、hub、交换机、电缆等因为这些都有可能导致故障。必须更换硬件才能正确诊断这些问题。
