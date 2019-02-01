MySQL5.6升级到5.7遇到的问题

18年我们借助MHA+mydumper+myloader将生产环境MySQL从5.6全部升级到了5.7版本，整体还算平稳。但是也遇到一些问题，主要是以下三类问题：
```
(1)主从复制问题
(2)range_optimizer_max_mem_size参数引起的性能问题
(3)SQL兼容问题
```


下面围绕这3个问题展开说明

### 1、主从复制问题
MySQL5.7到小于5.6.22的复制存在bug(bug 74683)，会导致复制中断，报错如下

```
2018-12-20 10:40:02 35878 [ERROR] Slave I/O: Found a Gtid_log_event or Previous_gtids_log_event when @@GLOBAL.GTID_MODE = OFF. Error_code: 1784
2018-12-20 10:40:02 35878 [ERROR] Slave I/O: Relay log write failure: could not queue event from master, Error_code: 1595
```
如果你的版本<5.6.23，建议你升级到>=5.6.23版本，因为这个bug是在5.6.23修复的，详细信息请见
https://bugs.mysql.com/bug.php?id=74683

### 2、range_optimizer_max_mem_size参数引起的性能问题
##### 【问题描述】
MySQL从5.6升级到5.7之后，开发反馈调度系统超时，和开发沟通后把问题SQL要了过来，SQL类似如下

```
select column1,column2 from tb123 where column1 in (3128611,3128612,3128613...这里省略30多万);
```
其中 column1 字段有索引idx_column1(column1)。

##### 【原因分析】
分别在5.6 5.7上查看了这条SQL的执行计划和执行时间，发现这条SQL在5.6版本使用了column1索引，执行时间2秒，但是在5.7版本全表扫描，执行时间是18秒。
同时在5.7显示了一个warnings

```
>show warnings;                                                                           
Warning | 3170 | Memory capacity of 8388608 bytes for 'range_optimizer_max_mem_size' exceeded. Range optimization was not done for this query.
```
从告警信息得知和参数 range_optimizer_max_mem_size 有关，查看官方文档得知：
这个参数是在mysql 5.7新增的，范围查询优化参数，这个参数限制范围查询优化使用的内存，默认值是8M，当使用内存超过8M则会放弃使用范围查询而采用其他方法比如用全表扫描来代替。而上面的SQL in里面的值太多，超出了8M，所以走了全表扫描。知道原因后，解决办法也很简单，就是增大这个值。
```
最小值: 0                   
最大值: 18446744073709551615
默认值: 8388608   
```
下面我们测试一下这个参数对查询的影响
##### 【测试】
使用 range_optimizer_max_mem_size 的默认值，即8M

```
>show variables like '%range_optimizer_max_mem_size%';
+------------------------------+---------+
| Variable_name                | Value   |
+------------------------------+---------+
| range_optimizer_max_mem_size | 8388608 |
+------------------------------+---------+
1 row in set (0.00 sec)

>show create table sbtest1\G
*************************** 1. row ***************************
       Table: sbtest1
Create Table: CREATE TABLE `sbtest1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `k` int(11) NOT NULL DEFAULT '0',
  `c` char(120) NOT NULL DEFAULT '',
  `pad` char(60) NOT NULL DEFAULT '',
  PRIMARY KEY (`id`),
  KEY `k_1` (`k`)
) ENGINE=InnoDB AUTO_INCREMENT=3000001 DEFAULT CHARSET=utf8
1 row in set (0.01 sec)

>desc select * from sbtest1 where k in (369193,434819,486940)\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: sbtest1
   partitions: NULL
         type: range
possible_keys: k_1
          key: k_1
      key_len: 4
          ref: NULL
         rows: 3
     filtered: 100.00
        Extra: Using index condition
1 row in set, 1 warning (0.00 sec)
```
从上面执行计划可以看到，使用到了二级索引 k_1。下面我们将range_optimizer_max_mem_size的值修改为2048测试

```
>set range_optimizer_max_mem_size=2048;                
Query OK, 0 rows affected (0.00 sec)

>show variables like '%range_optimizer_max_mem_size%';
+------------------------------+-------+
| Variable_name                | Value |
+------------------------------+-------+
| range_optimizer_max_mem_size | 2048  |
+------------------------------+-------+
1 row in set (0.00 sec)

>desc select * from sbtest1 where k in (369193,434819,486940)\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: sbtest1
   partitions: NULL
         type: ALL
possible_keys: k_1
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 2884885
     filtered: 30.00
        Extra: Using where
1 row in set, 2 warnings (0.00 sec)

>show warnings\G
*************************** 1. row ***************************
  Level: Warning
   Code: 3170
Message: Memory capacity of 2048 bytes for 'range_optimizer_max_mem_size' exceeded. Range optimization was not done for this query.
*************************** 2. row ***************************
  Level: Note
   Code: 1003
Message: /* select#1 */ select `sysbench`.`sbtest1`.`id` AS `id`,`sysbench`.`sbtest1`.`k` AS `k`,`sysbench`.`sbtest1`.`c` AS `c`,`sysbench`.`sbtest1`.`pad` AS `pad` from `sysbench`.`sbtest1` where (`sysbench`.`sbtest1`.`k` in (369193,434819,486940))
2 rows in set (0.00 sec)
```
从上面执行计划得知，走了全表扫描，没使用到二级索引k_1。

##### 【解决办法】
知道原因后，解决办法就很简单了，将 range_optimizer_max_mem_size 修改为100M后，SQL执行响应时间为2秒。

range_optimizer_max_mem_size修改为多大合适，可以参考官网计算公式
https://dev.mysql.com/doc/refman/5.7/en/range-optimization.html

### 3、SQL兼容性问题
有一套系统从5.6升级到5.7之后，开发反馈相同的SQL在5.6可以正常显示内容，但是升级后不显示内容了。
下面我用类似的例子展现一下当时的情况

```
表结构
CREATE TABLE `t_star` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(20) NOT NULL DEFAULT '' COMMENT '名字',
  `gender` tinyint(4) NOT NULL DEFAULT '0' COMMENT '0:男,1:女',
  `city` varchar(10) NOT NULL DEFAULT '' COMMENT '所在城市',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=6 DEFAULT CHARSET=utf8;

内容
>select * from t_star;
+----+-----------+--------+--------+
| id | name      | gender | city   |
+----+-----------+--------+--------+
|  1 | 姚明      |      0 | 上海   |
|  2 | 邓超      |      0 | 南昌   |
|  3 | 刘德华    |      0 | 香港   |
|  4 | 刘亦菲    |      1 | 香港   |
|  5 | 江疏影    |      1 | 上海   |
+----+-----------+--------+--------+
```
MySQL 5.6版本 

```
>select * from t_star where city = '上海' and gender in (x'ACED0005757200115B4C6A6176612E6C616E672E4C6F6E673B7DE10AB2BBBC632B0200007870000000017372000E6A6176612E6C616E672E4C6F6E673B8BE490CC8F23DF0200014A000576616C7565787200106A6176612E6C616E672E4E756D62657286AC951D0B94E08B02000078700000000000000001');
+----+-----------+--------+--------+
| id | name      | gender | city   |
+----+-----------+--------+--------+
|  5 | 江疏影    |      1 | 上海   |
+----+-----------+--------+--------+
```
MySQL 5.7版本

```
>select * from t_star where city = '上海' and gender in (x'ACED0005757200115B4C6A6176612E6C616E672E4C6F6E673B7DE10AB2BBBC632B0200007870000000017372000E6A6176612E6C616E672E4C6F6E673B8BE490CC8F23DF0200014A000576616C7565787200106A6176612E6C616E672E4E756D62657286AC951D0B94E08B02000078700000000000000001');
Empty set, 1 warning (0.00 sec)
>show warnings\G
*************************** 1. row ***************************
  Level: Warning
   Code: 1292
Message: Truncated incorrect BINARY value: 'x'aced0005757200115b4c6a6176612e6c616e672e4c6f6e673b7de10ab2bbbc632b0200007870000000017372000e6a6176612e6c616e672e4c6f6e673b8be4
```
从上面测试可以看出，同一条SQL在5.6可以正常显示结果，但是在5.7没显示任何信息且打印了一条告警信息。这个原因是，在5.6把gender的值转换成了1，而在5.7转换成了-1，SQL书写不规范导致的。

因此，核心业务系统从5.6升级到5.7，如果无法提前找出这种SQL，很可能会对业务造成影响，那么如何尽量避免这种问题，提前找到有问题的SQL呢？
这里提供一种思路，能从一定程度上减少这种事情的发生。
基本思路是：通过慢日志统计分析select，分别在5.6和5.7上运行，将运行结果输出到文件，对文件求MD5值，通过判断MD5值是否相同来判断相同SQL在5.6和5.7上执行结果是否相同。
下面是一种实现方法，可以参考。

假设要升级的MHA集群如下

```
192.168.1.10:3306 master 5.6
    ---192.168.1.20:3306 slave1 5.6
    ---192.168.1.30:3306 slave2	5.6
```
我们可以借助一台机器，在同一台机器上同时搭建5.6和5.7，如下所示

```
192.168.1.10:3306 master 5.6
    ---192.168.1.20:3306 slave1 5.6
    ---192.168.1.30:3306 slave2	5.6
    ---192.168.1.40:3306(5.6)---192.168.1.40:3307(5.7)
```
其中192.168.1.40是借助的机器，3306是5.6版本，3307是5.7版本，当检测SQL时，停止192.168.1.40:3306的复制即可，这样192.168.1.40:3306和192.168.1.40:3307数据是一致的。

下面是快速找出这种问题的简要步骤

```
1、将线上实例的慢日志的时间修改的很小，这里将long_query_time修改为0，收集1个小时慢日志
set global long_query_time=0;
收集期间注意观察服务器性能，别影响线上业务。

2、利用pt-query-digest分析慢日志并入表
pt-query-digest --user=username --password=password --limit=100% --charset=utf8 --progress percentage,1 --filter '$event->{fingerprint} =~ m/^select/i' --history h=aa.aa.aa.aa,P=3306,D=test,t=query_history --no-report /data/mysql3306/log/slow.log

其中query_history表结构如下
CREATE TABLE `query_history` (
  `checksum` bigint(20) unsigned NOT NULL,
  `sample` longtext NOT NULL,
  `db_min` varchar(100) NOT NULL DEFAULT '',
  `ts_min` d_time NOT NULL DEFAULT '0000-00-00 00:00:00',
  `ts_max` d_time NOT NULL DEFAULT '0000-00-00 00:00:00',
  `ts_cnt` float DEFAULT NULL,
  PRIMARY KEY (`checksum`,`ts_min`,`ts_max`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8

3、停止192.168.1.40:3306的复制，这样192.168.1.40:3306和192.168.1.40:3307数据是一致的。

4、执行下面脚本找出MD5值不同的SQL
#!/usr/bin/env python
# coding: utf8 -*-
check_hosts = ["192.168.1.40:3306","192.168.1.40:3307"]
manager_host="aa.aa.aa.aa:3306"
slow_log_to_file='/tmp/row.log'
import pymysql as connector
import traceback
import hashlib
red='\033[1;35m'
end='\033[0m'
lv1='\033[1;32m'
lv2='\033[0m'
def get_mysql_connection(server, db):
    try:
        dbconfig = {
            'user': 'username',
            'passwd': 'password',
            'charset': 'utf8mb4',
            'autocommit':True
        }
        host, port = server.split(':')
        port = int(port)
        dbconfig['host'] = host
        dbconfig['port'] = port
        dbconfig['db'] = db
        return connector.connect(**dbconfig)
    except Exception, e:
        print " get_mysql_connection() error : %s " % traceback.format_exc()
        raise e

def md5sum(filename, blocksize=65536):
    hash = hashlib.md5()
    with open(filename, "rb") as f:
        for block in iter(lambda: f.read(blocksize), b""):
            hash.update(block)
    return hash.hexdigest()

for check_host in check_hosts:
    print "%scheck mysql: %s %s" %(lv1,check_host,lv2)
    conn1 = get_mysql_connection(manager_host, 'test') 
    sql_num = """select count(*) from query_history"""
    cur1 = conn1.cursor()
    cur1.execute(sql_num)
    sql_num = cur1.fetchall()
    sql_num = sql_num[0][0]
    print "sql_num: %s" %(sql_num)
    i = 0
    while (i < sql_num):
        dbname = """select db_min from query_history limit %s,1""" %(i)
        sql = """select sample from query_history limit %s,1""" %(i)
        cur1.execute(dbname)
        dbname = cur1.fetchall()
        cur1.execute(sql)
        sql = cur1.fetchall()
	if dbname[0][0] <> 'information_schema':
            print "[%s] dbname: %s" %(i,dbname[0][0])  
            conn2 = get_mysql_connection(check_host, dbname[0][0]) 
            cur2 = conn2.cursor()
            cur2.execute(sql[0][0])
            rows = cur2.fetchall()
            conn2.close()
            with open(slow_log_to_file, 'w') as slow_log:
                for row in rows:
                    column=row[0]
                    print >> slow_log, "%s" %(column)         
            md5_value=md5sum(slow_log_to_file)
            print "%s" %(md5_value)
            i=i+1
        else:
            i=i+1
            pass
    conn1.close()


5、分析MD5值不同的SQL，推进改进
```

本文总结了MySQL从5.6升级到5.7过程中遇到的一些问题，希望对大家有帮助。








