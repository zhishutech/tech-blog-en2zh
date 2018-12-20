# MySQL 8.0 和 .FRM文件 Drop 怎么恢复表的DDL #


> 作者： Marco Tusa             
> 发布日期：2018-12-07        
> 关键词：Database Recovery, InnoDB, Insight for DBAs, JSON, MySQL  
> 适用范围：database recovery, MySQL8  
> 原文 https://www.percona.com/blog/2018/12/07/mysql-8-frm-drop-how-to-recover-table-ddl/


# *... or what I should keep in mind in case of disaster* # 
# *这篇文章或者可以叫灾难发生我应该记得做什么* #

To retrieve and maintain in SQL format the definition of all tables in a database, is a best practice that we all should adopt. To have that under version control is also another best practice to keep in mind.

以SQL的格式检索和维护数据库中所有表的定义是我们都应该采用的最佳实践。另一个需要记住的最佳实践是在版本控制下进行检索和维护。

While doing that may seem redundant, it can become a life saver in several situations. From the need to review what has historically changed in a table, to knowing who changed what and why… to when you need to recover your data and have your beloved MySQL instance not start…

以上的最佳实践看起来似乎是多余的，但它在几种情况下却是可以救命。 从想要知道谁改变了什么、为什么改而回顾表的历史变化，到需要要恢复数据甚至到MySQL实例不能启动等几种情况下，最佳实践就不多余了。

But let’s be honest, only a few do the right thing, and even fewer keep that information up to date. Given that’s the case, what can we do when we have the need to discover/recover the table structure?

但是说实话，只有很少一部分人做正确的事情，更少的人能够随时更新信息。鉴于此种情况，当需要发现/恢复表结构的时候，我们能做什么呢？

From the beginning, MySQL has used some external files to describe its internal structure.

开始，MySQL使用一些外部文件来描述自己的内部结构。

For instance, if I have a schema named windmills and a table named wmillAUTOINC1, on the file system I will see this:

例如，有一个名字为windmills的schema 和一个名字为wmillAUTOINC1的表，在文件系统中我们看到的是如下信息：

    -rw-r-----. 1 mysql mysql 8838 Mar 14 2018 wmillAUTOINC1.frm
    -rw-r-----. 1 mysql mysql   131072 Mar 14 2018 wmillAUTOINC1.ibd

The ibd file contains the data, while the frm file contains the structure information.

ibd文件包含数据，而frm文件则是包含表结构信息。

Putting aside ANY discussion about if this is safe, if it’s transactional and more… when we’ve experienced some major crash and data corruption this approach has been helpful. Being able to read from the frm file was the easiest way to get the information we need.

撇开任何关于这样做是不是安全，它是不是事务性等诸多问题暂不讨论。 当我们遇到一些重大的崩溃和数据损坏的时候，以上（外部文件存储内部结构）方法是很有用的。从frm文件中读取是我们获得我们想要的信息的最简单方法。

Simple tools like DBSake made the task quite trivial, and allowed us to script table definition when needed to run long, complex tedious data recovery:

像DBSake这样的简单工具使任务（从frm获取数据）变得非常简单，并且我们在需要运行冗长，复杂繁琐的数据恢复时，允许编写表定义脚本。

    [root@master1 windmills]# /opt/tools/dbsake frmdump wmillAUTOINC1.frm
    --
    -- Table structure for table `wmillAUTOINC1`
    -- Created with MySQL Version 5.7.20
    --
    CREATE TABLE `wmillAUTOINC1` (
      `id` bigint(11) NOT NULL AUTO_INCREMENT,
      `uuid` char(36) COLLATE utf8_bin NOT NULL,
      `millid` smallint(6) NOT NULL,
      `kwatts_s` int(11) NOT NULL,
      `date` date NOT NULL,
      `location` varchar(50) COLLATE utf8_bin NOT NULL,
      `active` tinyint(2) NOT NULL DEFAULT '1',
      `time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
      `strrecordtype` char(3) COLLATE utf8_bin NOT NULL,
      PRIMARY KEY (`id`),
      KEY `IDX_millid` (`millid`,`active`),
      KEY `IDX_active` (`id`,`active`)
    ) 
    
Of course, if the frm file was also corrupt, then we could try to get the information from the ibdata dictionary. If that is corrupted too (trust me I’ve seen all of these situations) … well a last resource was hoping the customer has a recent table definition stored somewhere, but as mentioned before, we are not so diligent, are we?

当然，如果frm文件也损坏了，我们可以尝试从ibdata 数据字典中获取信息。如果
数据字典也损坏了（相信我，我曾经遇到过这些状况）等状况发生，那么我们只能寄希望于客户有一个最近的表定义存储在某个地方，但是就像之前提到的那样，我们并不是那么勤奋，对吧？

Now, though, in MySQL8 we do not have FRM files, they were dropped. Even more interesting is that we do not have the same dictionary, most of the things that we knew have changed, including the dictionary location. So what can be done?

然而现在，在MySQL8.0中FRM已经被删除，所以没有FRM文件。更有趣的是，没有相同的数据字典，我们知道的大部分情况已经改变，这其中包含数据字典的位置。那么我们能做些什么呢？

Well Oracle have moved the FRM information—and more—to what is called Serialized Dictionary Information (SDI), the SDI is written INSIDE the ibd file, and represents the redundant copy of the information contained in the data dictionary.

Oracle 将FRM文件的信息及更多信息移动到叫做序列化字典信息（Serialized Dictionary Information，SDI），SDI被写在ibd文件内部，它是数据字典包含的信息的一个冗余副本。

The SDI is updated/modified by DDL operations on tables that reside in that tablespace. This is it: if you have one file per table normally, then you will have in that file ONLY the SDI for that table, but if you have multiple tables in a tablespace, the SDI information will refer to ALL of the tables.

表空间中表的DDL操作会更新修改SDI。这就是：如果正常一个表一个文件，那么这个文件就只有这张表的SDI，但是如果是多个表在一个表空间，那么SDI信息是表空间中所有的表的。

To extract this information from the IBD files, Oracle provides a utility called ibd2sdi. This application parses the SDI information and reports a JSON file that can be easily manipulated to extract and build the table definition.

为了从IBD文件中提取SDI信息，Oracle提供了一个应用程序  ibd2sdi。此应用程序解析SDI信息，并以JSON文件输出，该JSON文件可以方便地操作以提取和构建表定义。

One exception is represented by Partitioned tables. The SDI information is contained ONLY in the first partition, and if you drop it, it is moved to the next one. I will show that later.

分区表的SDI是个例外。分区表的SDI信息只存在于第一个分区，如果你删除第一个分区，SDI信息会移动到下一个分区。稍后展示。

But let’s see how it works. In the next examples I will look for the table’s name, attributes, and datatype starting from the dictionary tables.

让我们看以上动作是怎么实现的。在下个示例中，我将会在字典表中查找表的名字、属性、数据类型。

To obtain the info I will do this:

操作以下命令获得信息：

    /opt/mysql_templates/mysql-8P/bin/./ibd2sdi   /opt/mysql_instances/master8/data/mysql.ibd |jq  '.[]?|.[]?|.dd_object?|("------------------------------------"?,"TABLE NAME = ",.name?,"****",(.columns?|.[]?|(.name?,.column_type_utf8?)))'


The result will be something like:

结果显示如下：

    "------------------------------------"
    "TABLE NAME = "
    "tables"
    "****"
    "id"
    "bigint(20) unsigned"
    "schema_id"
    "bigint(20) unsigned"
    "name"
    "varchar(64)"
    "type"
    "enum('BASE TABLE','VIEW','SYSTEM VIEW')"
    "engine"
    "varchar(64)"
    "mysql_version_id"
    "int(10) unsigned"
    "row_format"
    "enum('Fixed','Dynamic','Compressed','Redundant','Compact','Paged')"
    "collation_id"
    "bigint(20) unsigned"
    "comment"
    "varchar(2048)"
    <snip>
    "------------------------------------"
    "TABLE NAME = "
    "tablespaces"
    "****"
    "id"
    "bigint(20) unsigned"
    "name"
    "varchar(259)"
    "options"
    "mediumtext"
    "se_private_data"
    "mediumtext"
    "comment"
    "varchar(2048)"
    "engine"
    "varchar(64)"
    "DB_TRX_ID"
    ""
    "DB_ROLL_PTR"
    ""

I cut the output for brevity, but if you run the above command yourself you’ll be able to see that this retrieves the information for ALL the tables residing in the IBD.

为了简洁起见，我精简了输出，但是如果自己运行以上命令，你将会检索到IBD文件中的所有表的SDI信息。

The other thing I hope you noticed is that I am NOT parsing ibdata, but mysql.ibd. Why? Because the dictionary was moved out from ibdata and is now in mysql.ibd.

我希望你注意到另一件事是：我不是在解析ibdata，而是在解析mysql.ibd。为什么呢？因为数据字典从ibdata移出，现在位于mysql.ibd中。

Look what happens if I try to parse ibdata:

看看如果我解析ibdata，会发生什么：

    [root@master1 ~]# /opt/mysql_templates/mysql-8P/bin/./ibd2sdi   /opt/mysql_instances/master8/data/ibdata1 |jq '.'
    [INFO] ibd2sdi: SDI is empty.


Be very careful here to not mess up your mysql.ibd file.

这里要非常小心，不要搞乱mysql.ibd文件。

Now what can I do to get information about my wmillAUTOINC1 table in MySQL8?

现在，如何在MySQL8.0中获得wmillAUTOINC1表的信息呢？

That is quite simple:

这很简单：

    /opt/mysql_templates/mysql-8P/bin/./ibd2sdi   /opt/mysql_instances/master8/data/windmills/wmillAUTOINC.ibd |jq '.'
    [
      "ibd2sdi",
      {
    "type": 1,
    "id": 1068,
    "object": {
      "mysqld_version_id": 80013,
      "dd_version": 80013,
      "sdi_version": 1,
      "dd_object_type": "Table",
      "dd_object": {
    "name": "wmillAUTOINC",
    "mysql_version_id": 80011,
    "created": 20180925095853,
    "last_altered": 20180925095853,
    "hidden": 1,
    "options": "avg_row_length=0;key_block_size=0;keys_disabled=0;pack_record=1;row_type=2;stats_auto_recalc=0;stats_sample_pages=0;",
    "columns": [
      {
    "name": "id",
    "type": 9,
    "is_nullable": false,
    "is_zerofill": false,
    "is_unsigned": false,
    "is_auto_increment": true,
    "is_virtual": false,
    "hidden": 1,
    "ordinal_position": 1,
    "char_length": 11,
    "numeric_precision": 19,
    "numeric_scale": 0,
    "numeric_scale_null": false,
    "datetime_precision": 0,
    "datetime_precision_null": 1,
    "has_no_default": false,
    "default_value_null": false,
    "srs_id_null": true,
    "srs_id": 0,
    "default_value": "AAAAAAAAAAA=",
    "default_value_utf8_null": true,
    "default_value_utf8": "",
    "default_option": "",
    "update_option": "",
    "comment": "",
    "generation_expression": "",
    "generation_expression_utf8": "",
    "options": "interval_count=0;",
    "se_private_data": "table_id=1838;",
    "column_key": 2,
    "column_type_utf8": "bigint(11)",
    "elements": [],
    "collation_id": 83,
    "is_explicit_collation": false
      },
    <SNIP>
    "indexes": [
      {
    "name": "PRIMARY",
    "hidden": false,
    "is_generated": false,
    "ordinal_position": 1,
    "comment": "",
    "options": "flags=0;",
    "se_private_data": "id=2261;root=4;space_id=775;table_id=1838;trx_id=6585972;",
    "type": 1,
    "algorithm": 2,
    "is_algorithm_explicit": false,
    "is_visible": true,
    "engine": "InnoDB",
    <Snip>
    ],
    "foreign_keys": [],
    "partitions": [],
    "collation_id": 83
      }
    }
      },
      {
    "type": 2,
    "id": 780,
    "object": {
      "mysqld_version_id": 80011,
      "dd_version": 80011,
      "sdi_version": 1,
      "dd_object_type": "Tablespace",
      "dd_object": {
    "name": "windmills/wmillAUTOINC",
    "comment": "",
    "options": "",
    "se_private_data": "flags=16417;id=775;server_version=80011;space_version=1;",
    "engine": "InnoDB",
    "files": [
      {
    "ordinal_position": 1,
    "filename": "./windmills/wmillAUTOINC.ibd",
    "se_private_data": "id=775;"
      }
    ]
      }
    }
      }
    ]
    

The JSON will contains:

这段JSON包含以下内容：

A section describing the DB object at high level
Array of columns and related information
Array of indexes
Partition information (not here but in the next example)
Table space information
That is a lot more detail compared to what we had in the FRM, and it is quite relevant and interesting information as well.

一段对DB对象的描述
列数组和相关信息
索引数组
分区信息（没有这以上展示的示例中，在下一个示例中）
表空间信息
这比我们在FRM文件中获得的信息更加详细，而且还有很有价值的信息。

Once you have extracted the SDI, any JSON parser tool script can generate the information for the SQL DDL.

一旦你提取了SDI，任何JSON解析工具执行都可以产生DDL的SQL信息。

I mention partitions, so let’s look at this a bit more, given they can be tricky.

刚刚我提到了分区，考虑到分区表的复杂，所以让我们详细的看一下

As mentioned, the SDI information is present ONLY in the first partition. All other partitions hold ONLY the tablespace information. Given that, then the first thing to do is to identify which partition is the first… OR simply try to access all partitions, and when you are able to get the details, extract them.

如上所述，分区表的SDI只出现在第一个分区。其他的分区仅仅保存表空间信息。既然如此，那么首先要做的就是确定哪个分区是第一个,或者简单地去尝试访问所有的分区，当你能够得到SDI详情时，提取它们。

The process is the same:

提取方法和过程与上面的办法相同：

    [root@master1 ~]# /opt/mysql_templates/mysql-8P/bin/./ibd2sdi   /opt/mysql_instances/master8/data/windmills/wmillAUTOINCPART#P#PT20170301.ibd |jq '.'
    [
      "ibd2sdi",
      {
    "type": 1,
    "id": 1460,
    "object": {
      "mysqld_version_id": 80013,
      "dd_version": 80013,
      "sdi_version": 1,
      "dd_object_type": "Table",
      "dd_object": {
    "name": "wmillAUTOINCPART",
    "mysql_version_id": 80013,
    "created": <strong>20181125110300</strong>,
    "last_altered": 20181125110300,
    "hidden": 1,
    "options": "avg_row_length=0;key_block_size=0;keys_disabled=0;pack_record=1;row_type=2;stats_auto_recalc=0;stats_sample_pages=0;",
    "columns": [<snip>
    	  "schema_ref": "windmills",
    "se_private_id": 18446744073709552000,
    "engine": "InnoDB",
    "last_checked_for_upgrade_version_id": 80013,
    "comment": "",
    "se_private_data": "autoinc=31080;version=2;",
    "row_format": 2,
    "partition_type": 7,
    "partition_expression": "to_days(`date`)",
    "partition_expression_utf8": "to_days(`date`)",
    "default_partitioning": 1,
    "subpartition_type": 0,
    "subpartition_expression": "",
    "subpartition_expression_utf8": "",
    "default_subpartitioning": 0,
       ],
    <snip>
    "foreign_keys": [],
    "partitions": [
      {
    "name": "PT20170301",
    "parent_partition_id": 18446744073709552000,
    "number": 0,
    "se_private_id": 1847,
    "description_utf8": "736754",
    "engine": "InnoDB",
    "comment": "",
    "options": "",
    "se_private_data": "autoinc=0;version=0;",
    "values": [
      {
    "max_value": false,
    "null_value": false,
    "list_num": 0,
    "column_num": 0,
    "value_utf8": "736754"
      }
    ],
    

The difference, as you can see, is that the section related to partitions and sub partitions will be filled with all the details you might need to recreate the partitions.

正如你所看到的，普通表和分区表的不同在于分区和子分区相关信息部分，这部分是分区的详情，你可能需要这部分去重新创建分区。

We will have:
Partition type
Partition expression
Partition values
…more
Same for sub partitions.

我们将得到：分区类型、分区表达式、分区值 等等。我们也可以获得子分区的这些信息。

Now again see what happens if I parse the second partition:

现在我们再解析第二个分区，看看会发生什么：


    [root@master1 ~]# /opt/mysql_templates/mysql-8P/bin/./ibd2sdi   /opt/mysql_instances/master8/data/windmills/wmillAUTOINCPART#P#PT20170401.ibd |jq '.'
    [
      "ibd2sdi",
      {
    "type": 2,
    "id": 790,
    "object": {
      "mysqld_version_id": 80011,
      "dd_version": 80011,
      "sdi_version": 1,
      "dd_object_type": "Tablespace",
      "dd_object": {
    "name": "windmills/wmillAUTOINCPART#P#PT20170401",
    "comment": "",
    "options": "",
    "se_private_data": "flags=16417;id=785;server_version=80011;space_version=1;",
    "engine": "InnoDB",
    "files": [
      {
    "ordinal_position": 1,
    "filename": "./windmills/wmillAUTOINCPART#P#PT20170401.ibd",
    "se_private_data": "id=785;"
      }
    ]
      }
    }
      }
    ]
    

I will get only the information about the tablespace, not the table.

我们只得到了表空间信息，而不是表信息。

As promised let me show you now what happens if I delete the first partition, and the second partition becomes the first:

如上所述，让我们来看一下如果我删除第一个分区，第二个分区成为第一个分区，会发生什么。


    (root@localhost) [windmills]>alter table wmillAUTOINCPART drop partition PT20170301;
    Query OK, 0 rows affected (1.84 sec)
    Records: 0  Duplicates: 0  Warnings: 0
    [root@master1 ~]# /opt/mysql_templates/mysql-8P/bin/./ibd2sdi   /opt/mysql_instances/master8/data/windmills/wmillAUTOINCPART#P#PT20170401.ibd |jq '.'|more
    [
      "ibd2sdi",
      {
    "type": 1,
    "id": 1461,
    "object": {
      "mysqld_version_id": 80013,
      "dd_version": 80013,
      "sdi_version": 1,
      "dd_object_type": "Table",
      "dd_object": {
    "name": "wmillAUTOINCPART",
    "mysql_version_id": 80013,
    "created": <strong>20181129130834</strong>,
    "last_altered": 20181129130834,
    "hidden": 1,
    "options": "avg_row_length=0;key_block_size=0;keys_disabled=0;pack_record=1;row_type=2;stats_auto_recalc=0;stats_sample_pages=0;",
    "columns": [
      {
    "name": "id",
    "type": 9,
    "is_nullable": false,
    "is_zerofill": false,
    "is_unsigned": false,
    "is_auto_increment": true,
    "is_virtual": false,
    "hidden": 1,
    "ordinal_position": 1,
    

As I mentioned before, each DDL updates the SDI, and here we go: I will have all the information on what’s NOW the FIRST partition. Please note the value of the attribute “created” between the first time I queried the other partition, and the one that I have now:

之前提到的，每一个DDL都会更新SDI，现在我们继续讨论：新成为第一的分区将会获得所有的SDI信息。请注意“created”属性的值，对比alter前查询的和alter后这次查询得到的值。

    /opt/mysql_instances/master8/data/windmills/wmillAUTOINCPART#P#PT20170301.ibd
           "created": 20181125110300,
    /opt/mysql_instances/master8/data/windmills/wmillAUTOINCPART#P#PT20170401.ibd
           "created": 20181129130834,
    

To be clear the second created is NOW (PT20170401) from when I dropped the other partition (PT20170301).
很清楚的的看到，第二个created 的时间是现在，也就是在我删除分区(PT20170301)时才创建的。


Conclusions
In the end, this solution is definitely more powerful than the FRM files. It will allow us to parse the file and identify the table definition more easily, providing us with much more detail and information.

结论
最终，MySQL8.0的解决方案肯定比FRM文件强大。它允许我们更容易的解析文件和表定义，并且提供了更多详细信息。

The problems will arise if and when the IBD file becomes corrupt.

如果或者当IBD文件崩溃，问题就会出现。

As for the manual:  *For InnoDB, an SDI record requires a single index page, which is 16KB in size by default.* However, SDI data is compressed to reduce the storage footprint.

如手册上所述，*对于InnoDB引擎，一个SDI记录需要一个索引页，默认大小16KB*。不过SDI可以被压缩以减少存储占用。

By which it means that for each table I have a page, if I associate record=table. Which means that in case of IBD corruption I should (likely) be able to read those pages. Unless I have bad (very bad) luck.

这就意味着 如果我认为SDI记录是表，那么每个表有一个page。也意味着，如果IBD损坏了，除非运气太差，不然我可以读取这些页获得信息。

I still wonder how the dimension of an IBD affects the SDI retrieval, but given I have not tried it yet I will have to let you know.


As an aside, I am working on a script to facilitate the generation of the SQL, it’s not yet ready but you can find it here

我仍然想知道从IBD是如何影响SDI，但是我必须让你知道我还没有尝试过。但是我正在编写一个脚本用来产生SQL，还没有完成，但是你能在这里找到这个脚本。

Last note but keep this in mind! It is stated in the manual but in a hidden place and in small letters:
*DDL operations take longer due to writing to storage, undo logs, and redo logs instead of .frm files*.

最后请注意以下内容在手册有说明，但是不明显：
*DDL操作要写入存储、undo 、redo 等文件，因此花费的时间比写入.frm文件时间长*。

References

参考文章

https://stedolan.github.io/jq/

https://dev.mysql.com/doc/refman/8.0/en/ibd2sdi.html

https://dev.mysql.com/doc/refman/8.0/en/serialized-dictionary-information.html

https://dev.mysql.com/doc/refman/8.0/en/data-dictionary-limitations.html
