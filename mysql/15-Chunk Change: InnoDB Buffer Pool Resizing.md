# Chunk Change: InnoDB Buffer Pool Resizing
原文：[Chunk Change: InnoDB Buffer Pool Resizing](https://www.percona.com/blog/2018/06/19/chunk-change-innodb-buffer-pool-resizing/)
译者：魏新平

----

从MySQL 5.7.5开始，我们可以动态修改InnoDB Buffer Pool的大小。这个新特性同时也引入了一个参数--innodb\_buffer\_pool\_chunk\_size，buffer pool会根据这个参数值的整数倍增加或减小。这个参数不是动态修改的，如果配置错误，可能会导致不想看到的结果。

>Since MySQL 5.7.5, we have been able to resize dynamically the InnoDB Buffer Pool. This new feature also introduced a new variable — innodb\_buffer\_pool\_chunk\_size — which defines the chunk size by which the buffer pool is enlarged or reduced. This variable is not dynamic and if it is incorrectly configured, could lead to undesired situations.

首先我们观察一下innodb\_buffer\_pool\_size , innodb\_buffer\_pool\_instances  and innodb\_buffer\_pool\_chunk\_size如何相互影响。buffer pool可以存放多个instance，每个instance由多个chunk组成。instance的数量范围和chunk的总数量范围分别为1-64，1-1000.

>Let’s see first how innodb\_buffer\_pool\_size , innodb\_buffer\_pool\_instances  and innodb\_buffer\_pool\_chunk\_size interact:
![avatar](https://www.percona.com/blog/wp-content/uploads/2018/04/Untitled-Diagram-2.jpg)
The buffer pool can hold several instances and each instance is divided into chunks. There is some information that we need to take into account: the number of instances can go from 1 to 64 and the total amount of chunks should not exceed 1000.

一个3G内存的服务器，128MB的chunk值，2GB的buffer pool，8个instance，那么每个instance就有2个chunk。
>So, for a server with 3GB RAM, a buffer pool of 2GB with 8 instances and chunks at default value (128MB) we are going to get 2 chunks per instance:
![avatar](https://www.percona.com/blog/wp-content/uploads/2018/04/bp8instances.png)
这意味着一共有16个chunks。
>This means that there will be 16 chunks.

本文只关注修改buffer pool大小的影响，所以不会阐述多个instance的好处。那为什么要修改buffer pool的大小呢？有多个理由，比如：
>I’m not going to explain the benefits of having multiple instances, I will focus on resizing operations. Why would you want to resize the buffer pool? Well, there are several reasons, such as:

如果数据库主机是虚拟机，可以根据需求动态的修改主机内存

如果数据库主机是物理机，可能会想减少数据库所占用的内存，让其他进程使用

一开始数据库的大小比可用内存小，根据规划，数据量会有巨大增长，这时，会需要增大buffer pool的大小
>on a virtual server you can add more memory dynamically

>for a physical server, you might want to reduce database memory usage to make way for other processes

>on systems where the database size is smaller than available RAM
if you expect a huge growth and want to increase the buffer pool on demand

##缩小buffer pool
让我们开始减小buffer pool
>Reducing the buffer pool

>Let’s start reducing the buffer pool:

```Shell

| innodb_buffer_pool_size | 2147483648 |
| innodb_buffer_pool_instances | 8     |
| innodb_buffer_pool_chunk_size | 134217728 |
mysql> set global innodb_buffer_pool_size=1073741824;
Query OK, 0 rows affected (0.00 sec)
mysql> show global variables like 'innodb_buffer_pool_size';
+-------------------------+------------+
| Variable_name           | Value      |
+-------------------------+------------+
| innodb_buffer_pool_size | 1073741824 |
+-------------------------+------------+
1 row in set (0.00 sec)
```
假如我们缩小buffer pool到1.5GB，buffer pool的大小不会改变并且会出现一个告警:
>If we try to decrease it to 1.5GB, the buffer pool will not change and a warning will be showed:

```Shell
mysql> show global variables like 'innodb_buffer_pool_size';
+-------------------------+------------+
| Variable_name           | Value      |
+-------------------------+------------+
| innodb_buffer_pool_size | 2147483648 |
+-------------------------+------------+
1 row in set (0.01 sec)
mysql> set global innodb_buffer_pool_size=1610612736;
Query OK, 0 rows affected, 1 warning (0.00 sec)
 
mysql> show warnings;
+---------+------+---------------------------------------------------------------------------------+
| Level   | Code | Message                                                                         |
+---------+------+---------------------------------------------------------------------------------+
| Warning | 1210 | InnoDB: Cannot resize buffer pool to lesser than chunk size of 134217728 bytes. |
+---------+------+---------------------------------------------------------------------------------+
1 row in set (0.00 sec)
```

##增加buffer pool

>Increasing the buffer pool

当我们把buffer pool从1GB增加到1.5GB时，1.5GB的值被认为是不合适，并且会被mysql调整为其他的值。
>When we try to increase the value from 1GB to 1.5GB, the buffer pool is resized but the requested innodb\_buffer\_pool\_size is considered to be incorrect and is truncated:

```Shell

mysql> set global innodb_buffer_pool_size=1610612736;
Query OK, 0 rows affected, 1 warning (0.00 sec)
mysql> show warnings;
+---------+------+-----------------------------------------------------------------+
| Level   | Code | Message                                                         |
+---------+------+-----------------------------------------------------------------+
| Warning | 1292 | Truncated incorrect innodb_buffer_pool_size value: '1610612736' |
+---------+------+-----------------------------------------------------------------+
1 row in set (0.00 sec)
mysql> show global variables like 'innodb_buffer_pool_size';
+-------------------------+------------+
| Variable_name           | Value      |
+-------------------------+------------+
| innodb_buffer_pool_size | 2147483648 |
+-------------------------+------------+
1 row in set (0.01 sec)
```

buffer pool最后的值为2GB。1.5GB的值被调整为了2GB，即使你多设置了1byte，比如设置成：1073741825，你还是会得到2GB大小的buffer pool。
>And the final size is 2GB. Yes! you intended to set the value to 1.5GB and you succeeded in setting it to 2GB. Even if you set 1 byte higher, like setting: 1073741825, you will end up with a buffer pool of 2GB.

```Shell
mysql> set global innodb_buffer_pool_size=1073741825;
Query OK, 0 rows affected, 1 warning (0.00 sec)
mysql> show global variables like 'innodb_buffer_pool_%size' ;
+-------------------------------+------------+
| Variable_name                 | Value      |
+-------------------------------+------------+
| innodb_buffer_pool_chunk_size | 134217728  |
| innodb_buffer_pool_size       | 2147483648 |
+-------------------------------+------------+
2 rows in set (0.01 sec)
```

##有趣的情景
增加配置文件当中的值
##Interesting scenarios
Increasing size in the config file

假如有一天，你突然想优化mysql的一些参数。由于服务器还有空闲内存，你想增加buffer pool的大小。在这个例子当中，我们将会在一台16个innodb\_buffer\_pool\_instances，2GB buffer pool的mysql上面做实验。我们会把buffer pool的大小增加到2.5GB。
>Let’s suppose one day you get up willing to change or tune some variables in your server, and you decide that as you have free memory you will increase the buffer pool. In this example, we are going to use a server with  innodb\_buffer\_pool\_instances = 16  and 2GB of buffer pool size which will be increased to 2.5GB

我们在配置文件当中配置如下参数值
>So, we set in the configuration file:

```Shell
innodb_buffer_pool_size = 2684354560
```

重启后我们发现
>But then after restart, we found:

```Shell
mysql> show global variables like 'innodb_buffer_pool_%size' ;
+-------------------------------+------------+
| Variable_name                 | Value      |
+-------------------------------+------------+
| innodb_buffer_pool_chunk_size | 134217728  |
| innodb_buffer_pool_size       | 4294967296 |
+-------------------------------+------------+
2 rows in set (0.00 sec)
```

错误日志发现如下报错
>And the error log says:

```Shell
2018-05-02T21:52:43.568054Z 0 [Note] InnoDB: Initializing buffer pool, total size = 4G, instances = 16, chunk size = 128M
```

由于instance和chunk的影响，配置文档当中的2.5GB没有生效，重启后buffer pool变成了4GB。日志信息并没有告诉我们chunk的数量，但是这个数量值对于理解为何会出现这种现象有很大的帮助。

>So, after we have set innodb\_buffer\_pool\_size in the config file to 2.5GB, the database gives us a 4GB buffer pool, because of the number of instances and the chunk size. What the message doesn’t tell us is the number of chunks, and this would be useful to understand why such a huge difference.

让我们观察一下4GB是如何被算出来的
>Let’s take a look at how that’s calculated.

增加instance和chunk的值

修改instance或者chunk size需要重启数据库并且需要考虑buffer pool的上限。比如配置成如下的值
>Increasing instances and chunk size

>Changing the number of instances or the chunk size will require a restart and will take into consideration the buffer pool size as an upper limit to set the chunk size. For instance, with this configuration:

```Shell
innodb_buffer_pool_size = 2147483648
innodb_buffer_pool_instances = 32
innodb_buffer_pool_chunk_size = 134217728
```

我们得到如下的chunk大小
>We get this chunk size:

```Shell

mysql> show global variables like 'innodb_buffer_pool_%size' ;
+-------------------------------+------------+
| Variable_name                 | Value      |
+-------------------------------+------------+
| innodb_buffer_pool_chunk_size | 67108864   |
| innodb_buffer_pool_size       | 2147483648 |
+-------------------------------+------------+
2 rows in set (0.00 sec)
```
那么，如何计算innodb\_buffer\_pool\_chunk\_size的大小呢。innodb\_buffer\_pool\_size除以innodb\_buffer\_pool\_instances，得到的值再根据1MB的整数倍四舍五入。

>However, we need to understand how this is really working. To get the innodb\_buffer\_pool\_chunk_size it will make this calculation: innodb\_buffer\_pool\_size / innodb\_buffer\_pool\_instances with the result rounded to a multiple of 1MB.


在上述例子当中，计算方式为2147483648 / 32 = 67108864，而67108864除以1048576等于0，刚好是1MB的整数倍。每个instance刚好一个chunk。
>In our example, the calculation will be 2147483648 / 32 = 67108864 which 67108864%1048576=0, no rounding needed. The number of chunks will be one chunk per instance.

那什么情况下instance会有多个chunk呢？当想要的innodb\_buffer\_pool\_size的大小和配置文件当中配置的值相差比1MB大或者等于1MB的时候。
>When does it consider that it needs to use more chunks per instance? When the difference between the required size and the innodb\_buffer\_pool\_size configured in the file is greater or equal to 1MB.

这就是为什么假如你设置innodb\_buffer\_pool\_size为1GB+1MB-1B，你将会得到1GB的buffer pool。
>That is why, for instance, if you try to set the innodb_buffer_pool_size equal to 1GB + 1MB – 1B you will get 1GB of buffer pool:

```Shell
innodb_buffer_pool_size = 1074790399
innodb_buffer_pool_instances = 16
innodb_buffer_pool_chunk_size = 67141632
2018-05-07T09:26:43.328313Z 0 [Note] InnoDB: Initializing buffer pool, total size = 1G, instances = 16, chunk size = 64M
```
但是假如你设置innodb_buffer_pool_size为1GB+1MB，你将会得到2GB的buffer pool。
>But if you set the innodb_buffer_pool_size equals to 1GB + 1MB you will get 2GB of buffer pool:

```Shell
innodb_buffer_pool_size = 1074790400
innodb_buffer_pool_instances = 16
innodb_buffer_pool_chunk_size = 67141632
2018-05-07T09:25:48.204032Z 0 [Note] InnoDB: Initializing buffer pool, total size = 2G, instances = 16, chunk size = 64M
```
这是因为它认为两个chunk刚好。我们可以认为Innodb Buffer pool是这样计算的。
>This is because it considers that two chunks will fit. We can say that this is how the InnoDB Buffer pool size is calculated:

```Shell
determine_best_chunk_size{
  if innodb_buffer_pool_size / innodb_buffer_pool_instances < innodb_buffer_pool_chunk_size
  then
    innodb_buffer_pool_chunk_size = roundDownMB(innodb_buffer_pool_size / innodb_buffer_pool_instances)
  fi
}
determine_amount_of_chunks{
  innodb_buffer_amount_chunks_per_instance = roundDown(innodb_buffer_pool_size / innodb_buffer_pool_instances / innodb_buffer_pool_chunk_size)
  if innodb_buffer_amount_chunks_per_instance * innodb_buffer_pool_instances * innodb_buffer_pool_chunk_size - innodb_buffer_pool_size > 1024*1024
  then
    innodb_buffer_amount_chunks_per_instance++
  fi
}
determine_best_chunk_size
determine_amount_of_chunks
innodb_buffer_pool_size = innodb_buffer_pool_instances * innodb_buffer_pool_chunk_size * innodb_buffer_amount_chunks_per_instance
determine_best_chunk_size{
  if innodb_buffer_pool_size / innodb_buffer_pool_instances < innodb_buffer_pool_chunk_size
  then
    innodb_buffer_pool_chunk_size = roundDownMB(innodb_buffer_pool_size / innodb_buffer_pool_instances)
  fi
}
determine_amount_of_chunks{
  innodb_buffer_amount_chunks_per_instance = roundDown(innodb_buffer_pool_size / innodb_buffer_pool_instances / innodb_buffer_pool_chunk_size)
 
  if innodb_buffer_amount_chunks_per_instance * innodb_buffer_pool_instances * innodb_buffer_pool_chunk_size - innodb_buffer_pool_size > 1024*1024
  then
    innodb_buffer_amount_chunks_per_instance++
  fi
}
 
determine_best_chunk_size
determine_amount_of_chunks
 
innodb_buffer_pool_size = innodb_buffer_pool_instances * innodb_buffer_pool_chunk_size * innodb_buffer_amount_chunks_per_instance
```

那什么才是最合适的配置呢？
为了分析最好的配置，你需要知道chunk有1000个数量限制。在我们的例子当中，每个instance不能超过62个chunk。
>What is the best setting?
>In order to analyze the best setting you will need to know that there is a upper limit of 1000 chunks. In our example with 16 instances, we can have no more than 62 chunks per instance.

另一个需要考虑的事情是每个chunk占多大百分比。继续上面的例子，每个chunk每个instance代表了1.61%的大小，我们只能按照这个值的整数倍来修改innodb buffer pool的大小。
>Another thing to consider is what each chunk represents in percentage terms. Continuing with the example, each chunk per instance represent 1.61%, which means that we can increase or decrease the complete buffer pool size in multiples of this percentage.

从管理的角度来说，我认为你可能想要考虑最少2%到%5来增加或者减少buffer。我做了一些测试来验证小chunk值对数据库的影响，但是没有找到任何重要的东西。
>From a management point of view, I think that you might want to consider at least a range of 2% to 5% to increase or decrease the buffer. I performed some tests to see the impact of having small chunks and I found no issues but this is something that needs to be thoroughly tested.
