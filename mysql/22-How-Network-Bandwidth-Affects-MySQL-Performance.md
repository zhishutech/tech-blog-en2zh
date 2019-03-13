# 网络带宽是如何影响MySQL性能的

作者：[Vadim Tkachenko](https://www.percona.com/blog/author/vadim/)

发布时间：2018-10-22

标签：[network], [network performance]

文章原文：[How Network Bandwidth Affects MySQL Performance](https://www.percona.com/blog/2019/02/19/how-network-bandwidth-affects-mysql-performance/)

*Network is a major part of a database infrastructure. However, often performance benchmarks are done on a local machine, where a client and a server are collocated – I am guilty myself. This is done to simplify the setup and to exclude one more variable (the networking part), but with this we also miss looking at how network affects performance.*

网络是数据库基础架构的主要部分。但是，通常性能基准测试是在本地计算机上完成的，客户端和服务器同一台机器上。这样做是为了简化设置并排除网络部分的因素，但是我们也错过了查看网络如何影响MySQL性能

*The network is even more important for clustering products like [Percona XtraDB Cluster](https://www.percona.com/software/mysql-database/percona-xtradb-cluster) and [MySQL Group Replication](https://dev.mysql.com/doc/refman/8.0/en/group-replication.html). Also, we are working on our [Percona XtraDB Cluster Operator](https://www.percona.com/blog/2019/01/18/percona-xtradb-cluster-operator-early-access-0-2-0-release-is-now-available/) for Kubernetes and OpenShift, where network performance is critical for overall performance.*

对于像[Percona XtraDB Cluster](https://www.percona.com/software/mysql-database/percona-xtradb-cluster)和[MySQL Group Replication](https://dev.mysql.com/doc/refman/8.0/en/group-replication.html)这样的集群产品来说，网络更为重要。此外，我们正在为Kubernetes和OpenShift开发pxc operator，其中网络性能对整体性能至关重要。

*In this post, I will look into networking setups. These are simple and trivial, but are a building block towards understanding networking effects for more complex setups.*

在这篇文章中，我将深入探究网络设置。这些都是简单而微不足道的，但它们是了解更复杂设置的网络影响的基石。

## 设置

*I will use two bare-metal servers, connected via a dedicated 10Gb network. I will emulate a 1Gb network by changing the network interface speed with ethtool -s eth1 speed 1000 duplex full autoneg off command*

我将使用两台裸机服务器，通过专用的10Gb的网络连接。我将通过`ethtool -s eth1 speed 1000 duplex full autoneg off`命令来模拟1Gb的网络

![network test topology](https://www.percona.com/blog/wp-content/uploads/2019/02/network-test-topology.png)

*I will run a simple benchmark:*

我将会运行一个简单的压测

`sysbench oltp_read_only --mysql-ssl=on --mysql-host=172.16.0.1 --tables=20 --table-size=10000000 --mysql-user=sbtest --mysql-password=sbtest --threads=$i --time=300 --report-interval=1 --rand-type=pareto`

*This is run with the number of threads varied from 1 to 2048. All data fits into memory – innodb_buffer_pool_size is big enough – so the workload is CPU-intensive in memory: there is no IO overhead.*

压测线程数将会从1增长到2048。所有的数据全部都存在内存当中-innodb_buffer_pool_size足够大-所以工作的负载时在cpu密集型：没有IO的开销

*Operating System: Ubuntu 16.04*

### Benchmark N1. Network bandwidth

*In the first experiment I will compare 1Gb network vs 10Gb network.*

在第一个实验当中，我将对比1Gb网络和10Gb网络

![1gb vs 10gb network](https://www.percona.com/blog/wp-content/uploads/2019/02/1gb-vs-10gb-network.png)

| **threads/throughput** | **1Gb network** | **10Gb network** |
| ---------------------- | --------------- | ---------------- |
| 1                      | 326.13          | 394.4            |
| 4                      | 1143.36         | 1544.73          |
| 16                     | 2400.19         | 5647.73          |
| 32                     | 2665.61         | 10256.11         |
| 64                     | 2838.47         | 15762.59         |
| 96                     | 2865.22         | 17626.77         |
| 128                    | 2867.46         | 18525.91         |
| 256                    | 2867.47         | 18529.4          |
| 512                    | 2867.27         | 17901.67         |
| 1024                   | 2865.4          | 16953.76         |
| 2048                   | 2761.78         | 16393.84         |

*Obviously the 1Gb network performance is a bottleneck here, and we can improve our results significantly if we move to the 10Gb network.*

很显然，1Gb网络性能是这里的瓶颈，如果我们迁移到10Gb网络，我们可以显着改善我们的结果。

*To see that 1Gb network is bottleneck we can check the network traffic chart in PMM:*

我们可以从PMM的网络流量图中可以看到1Gb的网络是瓶颈

![network traffic in PMM](https://www.percona.com/blog/wp-content/uploads/2019/02/network-traffic-in-PMM.png)

*We can see we achieved 116MiB/sec (or 928Mb/sec)  in throughput, which is very close to the network bandwidth.*

我们可以看到outbound的网络流量已经达到了116MB/s(或者928Mb/s)，已经非常接近网络带宽了

*But what we can do if the our network infrastructure is limited to 1Gb?*

但是网络设施在1Gb的限制下，我们还能做什么呢？

### Benchmark N2. Protocol compression

*There is a feature in MySQL protocol whereby you can see the compression for the network exchange between client and server: --mysql-compression=on  for sysbench.*

在MySQL协议中有一个特性，你可以通过压缩客户端和服务端的网络交换，sysbench可以打开`--mysql-compression`

*Let’s see how it will affect our results.*

让我们来看下它会不会影响我们的结果

![1gb network with compression protocol](https://www.percona.com/blog/wp-content/uploads/2019/02/1gb-network-with-compression-protocol.png)

| threads/throughput | 1Gb network | 1Gb with compression protocol |
| ------------------ | ----------- | ----------------------------- |
| 1                  | 326.13      | 198.33                        |
| 4                  | 1143.36     | 771.59                        |
| 16                 | 2400.19     | 2714                          |
| 32                 | 2665.61     | 3939.73                       |
| 64                 | 2838.47     | 4454.87                       |
| 96                 | 2865.22     | 4770.83                       |
| 128                | 2867.46     | 5030.78                       |
| 256                | 2867.47     | 5134.57                       |
| 512                | 2867.27     | 5133.94                       |
| 1024               | 2865.4      | 5129.24                       |
| 2048               | 2761.78     | 5100.46                       |

*Here is an interesting result. When we use all available network bandwidth, the protocol compression actually helps to improve the result.*

实验结果很有意思，当我们用尽了所有的网络带宽的时候，压缩协议实际上是帮我们提高了吞吐量

![10g network with compression protocol](https://www.percona.com/blog/wp-content/uploads/2019/02/10g-network-with-compression-protocol.png)

| threads/throughput | 10Gb     | 10Gb with compression |
| ------------------ | -------- | --------------------- |
| 1                  | 394.4    | 216.25                |
| 4                  | 1544.73  | 857.93                |
| 16                 | 5647.73  | 3202.2                |
| 32                 | 10256.11 | 5855.03               |
| 64                 | 15762.59 | 8973.23               |
| 96                 | 17626.77 | 9682.44               |
| 128                | 18525.91 | 10006.91              |
| 256                | 18529.4  | 9899.97               |
| 512                | 17901.67 | 9612.34               |
| 1024               | 16953.76 | 9270.27               |
| 2048               | 16393.84 | 9123.84               |

*But this is not the case with the 10Gb network. The CPU resources needed for compression/decompression are a limiting factor, and with compression the throughput actually only reach half of what we have without compression.*

但是在10Gb网络下并没有起到作用，压缩和解压缩需要的cpu资源变成了瓶颈，压缩下的性能实际上只有未压缩的一半

*Now let’s talk about protocol encryption, and how using SSL affects our results.*

现在，我们来讨论下协议加密和使用SSL是否影响我们的结果

### Benchmark N3. Network encryption

![1gb network and 1gb with SSL](https://www.percona.com/blog/wp-content/uploads/2019/02/1gb-network-and-1gb-with-SSL.png)

| threads/throughput | 1Gb network | 1Gb SSL |
| ------------------ | ----------- | ------- |
| 1                  | 326.13      | 295.19  |
| 4                  | 1143.36     | 1070    |
| 16                 | 2400.19     | 2351.81 |
| 32                 | 2665.61     | 2630.53 |
| 64                 | 2838.47     | 2822.34 |
| 96                 | 2865.22     | 2837.04 |
| 128                | 2867.46     | 2837.21 |
| 256                | 2867.47     | 2837.12 |
| 512                | 2867.27     | 2836.28 |
| 1024               | 2865.4      | 1830.11 |
| 2048               | 2761.78     | 1019.23 |

![10gb network and 10gb with SSL](https://www.percona.com/blog/wp-content/uploads/2019/02/10gb-network-and-10gb-with-SSL.png)

| **threads/throughput** | **10Gb** | **10Gb SSL** |
| ---------------------- | -------- | ------------ |
| 1                      | 394.4    | 359.8        |
| 4                      | 1544.73  | 1417.93      |
| 16                     | 5647.73  | 5235.1       |
| 32                     | 10256.11 | 9131.34      |
| 64                     | 15762.59 | 8248.6       |
| 96                     | 17626.77 | 7801.6       |
| 128                    | 18525.91 | 7107.31      |
| 256                    | 18529.4  | 4726.5       |
| 512                    | 17901.67 | 3067.55      |
| 1024                   | 16953.76 | 1812.83      |
| 2048                   | 16393.84 | 1013.22      |

*For the 1Gb network, SSL encryption shows some penalty – about 10% for the single thread – but otherwise we hit the bandwidth limit again. We also see some scalability hit on a high amount of threads, which is more visible in the 10Gb network case.*

对于1Gb网络，SSL加密显示了一些损失 - 单线程约为10％ - 但是其他情况下我们再次达到带宽限制。在10Gb网络中，我们看到在大量线程下性能损失更加明显。

*With 10Gb, the SSL protocol does not scale after 32 threads. Actually, it appears to be a scalability problem in OpenSSL 1.0, which MySQL currently uses.*

在10Gb网络下，使用SSL协议在32线程后，吞吐量并没有增加。实际上，它似乎是MySQL目前使用的OpenSSL 1.0中的可伸缩性问题。

*In our experiments, we saw that OpenSSL 1.1.1 provides much better scalability, but you need to have a special build of MySQL from source code linked to OpenSSL 1.1.1 to achieve this. I don’t show them here, as we do not have production binaries.*

在我们的实验中，我们看到OpenSSL 1.1.1提供了更好的可伸缩性，但是您需要从链接到OpenSSL 1.1.1的源代码中获得特殊的MySQL构建来实现这一点。我没有在这里展示它们，因为我们没有生产版本的二进制文件。

## Conclusions

1. Network performance and utilization will affect the general application throughput.
2. Check if you are hitting network bandwidth limits
3. Protocol compression can improve the results if you are limited by network bandwidth, but also can make things worse if you are not
4. SSL encryption has some penalty (~10%) with a low amount of threads, but it does not scale for high concurrency workloads.

## 结论

1. 网络性能和利用率将影响一般应用程序吞吐量
2. 检查您是否达到了网络带宽限制
3. 如果受到网络带宽的限制，协议压缩可以改善结果，但如果不是，则会使事情变得更糟
4. SSL加密在线程数量较少的情况下会有一些损失（约10％），但对于高并发工作负载，性能并不会有所增长