# 嗦一嗦 MySQL 8.0的新特性 What’s New in MySQL 8.0? (Generally Available)

原文链接：https://mysqlserverteam.com/whats-new-in-mysql-8-0-generally-available

原文作者：Geir Hoydalsvik （Oracle官方开发工程师）

翻译：张锐志

[TOC]

非常高兴的向大家宣布MySQL 8.0 GA版本发布，MySQL 8.0是一个得到全面增强且极具吸引力的新版本。不限于下面几点：

We proudly announce General Availability of MySQL 8.0. [Download now!](http://dev.mysql.com/downloads/mysql/) MySQL 8.0 is an extremely exciting new version of the world’s most popular open source database with improvements across the board. Some key enhancements include:

**SQL方面**：窗口函数，公共表达式，NOWAIT, SKIPLOCKED, 降序索引，分组，正则表达式，字符集，CBO优化模式，直方图

1. **SQL** Window functions, Common Table Expressions, NOWAIT and SKIP LOCKED, Descending Indexes, Grouping, Regular Expressions, Character Sets, Cost Model, and Histograms.

**对JSON的支持**：扩充语法，新函数，排序增强，JSON列部分更新。基于JSON表的特性，可以调用SQL语句处理JSON数据。

2. **JSON** Extended syntax, new functions, improved sorting, and partial updates. With JSON table functions you can use the SQL machinery for JSON data.

**对地理信息系统的支持** 空间引用系统（SRS），包括SRS空间数据类型，空间索引，空间函数

3. **GIS** Geography support. Spatial Reference Systems (SRS), as well as SRS aware spatial datatypes,  spatial indexes,  and spatial functions.

**可靠性**：DDL语句支持原子性和崩溃安全恢复（元信息数据被存在了一个基于InnoDB的单独事务性数据字典中）。

4. **Reliability** DDL statements have become atomic and crash safe, meta-data is stored in a single, transactional data dictionary. Powered by InnoDB! 

**可观察性**：对P_S,I_S,配置参数，错误日志的记录有显著增强

5. **Observability** Significant enhancements to Performance Schema, Information Schema, Configuration Variables, and Error Logging.

**可管理性**：远程管理，Undo表空间管理，快速DDL

6. **Manageability** Remote management, Undo tablespace management, and new instant DDL.

**安全性**：OpenSSL的改进，新的默认验证方式，SQL角色权限，分解super权限，密码强度提升等等

7. **Security** OpenSSL improvements, new default authentication, SQL Roles, breaking up the super privilege, password strength, and more.

**性能**：InnoDB在读/写负载，高IO负载，热数据高并发竞争等场景表现更好。新增的资源组特性给用户在特定负载和特定硬件情况下将用户线程映射到指定的CPU上的可选项

8. **Performance** InnoDB is significantly better at Read/Write workloads, IO bound workloads, and high contention “hot spot” workloads. Added Resource Group feature to give users an option optimize for specific workloads on specific hardware by mapping user threads to CPUs.

以上是8.0版本的部分亮点，我（原文作者）推荐您仔细阅读GA版本前几个版本的发布信息，甚至这些特性和实现方法的的项目日志。或者您可以选择直接在Github上阅读源码。

The above represents some of the highlights and I encourage you to further drill into the complete series of Milestone blog posts—[8.0.0](http://mysqlserverteam.com/the-mysql-8-0-0-milestone-release-is-available/), [8.0.1](http://mysqlserverteam.com/the-mysql-8-0-1-milestone-release-is-available/), [8.0.2](http://mysqlserverteam.com/the-mysql-8-0-2-milestone-release-is-available/), [8.0.3](https://mysqlserverteam.com/the-mysql-8-0-3-release-candidate-is-available/), and [8.0.4](https://mysqlserverteam.com/the-mysql-8-0-4-release-candidate-is-available/) —and even further down in to the individual [worklogs](http://dev.mysql.com/worklog/) with their specifications and implementation details. Or perhaps you prefer to just look at the source code at [github.com/mysql](https://github.com/mysql).

## 面向开发人员的特性  **Developer features**

MySQL 8.0应面向MySQL开发人员的需求，带来了SQL，JSON，正则表达式，地理信息系统等方面的特性，因为很多开发人员有存储EmoJi表情的需求，在新版本中UTF8MB4成为默认的字符集。除此之外，还有对Binary数据类型按位操作，和对IPV6和UUID函数的改进。

MySQL Developers want new features and MySQL 8.0 delivers many new and much requested features in areas such as SQL, JSON, Regular Expressions, and GIS. Developers also want to be able to store Emojis, thus UTF8MB4 is now the default character set in 8.0. Finally there are improvements in Datatypes, with bit-wise operations on BINARY datatypes and improved IPv6 and UUID functions.

### SQL

###### 窗口函数 **Window Functions**

MySQL 8.0带来了标准SQL的窗口函数功能，窗口函数与分组聚合函数相类似的是都提供了对一组行数据的统计计算。但与分组聚合函数将多行合并成一行不同是窗口函数会在结果结果集中展现每一行的聚合。

MySQL 8.0 delivers SQL window functions.   Similar to grouped aggregate functions, window functions perform some calculation on a set of rows, e.g. `COUNT` or `SUM`. But where a grouped aggregate collapses this set of rows into a single row, a window function will perform the aggregation for each row in the result set.

窗口函数有两种使用方式，首先是常规的SQL聚合功能函数和特殊的窗口函数。常规的聚合功能函数如：`COUNT`,`SUM`等函数。而窗口函数专有的则是`RANK`, `DENSE_RANK`, `PERCENT_RANK`, `CUME_DIST`, `NTILE`, `ROW_NUMBER`, `FIRST_VALUE`, `LAST_VALUE`, `NTH_VALUE`, `LEAD `and `LAG`等函数

Window functions come in two flavors: SQL aggregate functions used as window functions and specialized window functions. This is the set of aggregate functions in MySQL that support windowing: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `BIT_OR`, `BIT_AND`, `BIT_XOR`, `STDDEV_POP` (and its synonyms `STD`, `STDDEV`), `STDDEV_SAMP`, `VAR_POP` (and its synonym `VARIANCE`) and `VAR_SAMP`. The set of specialized window functions are: `RANK`, `DENSE_RANK`, `PERCENT_RANK`, `CUME_DIST`, `NTILE`, `ROW_NUMBER`, `FIRST_VALUE`, `LAST_VALUE`, `NTH_VALUE`, `LEAD `and `LAG`

对窗口函数的支持上，是用户呼声比较频繁。窗口函数早在SQL2003规范中就成为了标准SQL的一部分。

Support for window functions (a.k.a. analytic functions) is a frequent user request. Window functions have long been part of standard SQL (SQL 2003). See blog post by Dag Wanvik [here](http://mysqlserverteam.com/mysql-8-0-2-introducing-window-functions/) as well as blog post by Guilhem Bichot [here](https://mysqlserverteam.com/row-numbering-ranking-how-to-use-less-user-variables-in-mysql-queries/).

###### 公用表表达式（CTE）

###### **Common Table Expression**

MySQL 8.0 带来了支持递归的公用表表达式的功能。非递归的公用表表达式由于允许由form子句派生的临时表的原因可以被多次引用，因而被解释为改进型的派生表（from子句中的临时表）。而递归的公用表表达式则由一组原始住居，经过处理后得到新的一组数据，再被带入处理得到更多的新数据，循环往复直到再也无法产生更多新数据为止。公用表达式也是一个用户呼声频繁的SQL功能。

MySQL 8.0 delivers [Recursive] Common Table Expressions (CTEs).  Non-recursive CTEs can be explained as “improved derived tables” as it allow the derived table to be referenced more than once. A recursive CTE is a set of rows which is built iteratively: from an initial set of rows, a process derives new rows, which grow the set, and those new rows are fed into the process again, producing more rows, and so on, until the process produces no more rows. CTE is a commonly requested SQL feature, see for example feature request [16244 ](https://bugs.mysql.com/bug.php?id=16244)and [32174 ](https://bugs.mysql.com/bug.php?id=32174). See blog posts by Guilhem Bichot [here](http://mysqlserverteam.com/mysql-8-0-labs-recursive-common-table-expressions-in-mysql-ctes/), [here](http://mysqlserverteam.com/mysql-8-0-labs-recursive-common-table-expressions-in-mysql-ctes-part-two-how-to-generate-series/), [here](http://mysqlserverteam.com/mysql-8-0-labs-recursive-common-table-expressions-in-mysql-ctes-part-three-hierarchies/), and [here](http://mysqlserverteam.com/mysql-8-0-1-recursive-common-table-expressions-in-mysql-ctes-part-four-depth-first-or-breadth-first-traversal-transitive-closure-cycle-avoidance/).

###### 新的NOWAIT、SKIP LOCKED选项

###### NOWAIT and SKIP LOCKED**

MySQL 8.0 给SQL的上锁子句带来了`NOWAIT`和`SKIP LOCKED`两个可选项。在原来的版本中，当行数据被`UPDATE`或者`SELECT ... FOR UPDATE`语句上锁后，其他的事务需要等待锁释放才能访问这行数据。但在某些场景下，有需要马上获得（不等待锁）数据的需求。使用`NOWAIT`参数后如果请求的数据中包括了被锁住的行，将马上会收到查询失败的报错信息。使用`SKIP LOCKED`参数后，返回的数据将会跳过被锁住的行。

MySQL 8.0 delivers `NOWAIT` and `SKIP LOCKED` alternatives in the SQL locking clause. Normally, when a row is locked due to an `UPDATE` or a `SELECT ... FOR UPDATE`, any other transaction will have to wait to access that locked row. In some use cases there is a need to either return immediately if a row is locked or ignore locked rows. A locking clause using `NOWAIT` will never wait to acquire a row lock. Instead, the query will fail with an error. A locking clause using `SKIP LOCKED` will never wait to acquire a row lock on the listed tables. Instead, the locked rows are skipped and not read at all. NOWAIT and SKIP LOCKED are frequently requested SQL features. See for example feature request [49763 ](https://bugs.mysql.com/bug.php?id=49763). We also want to say thank you to *Kyle Oppenheim* for his code contribution! See blog post by Martin Hansson [here](https://mysqlserverteam.com/mysql-8-0-1-using-skip-locked-and-nowait-to-handle-hot-rows/).

###### 降序索引

###### Descending Indexes**

MySQL 8.0 带来了对降序索引的支持。在 8.0降序索引中，数据被倒序组织，正向查找。而在之前的版本中，虽然支持创建降序排列的索引，但其实现方式是通过创建常见的正序索引，然后进行反向查找来实现的。一方面来说，正序查找要比逆序查找更快，另一方面来说，真正的降序索引在复合的order by语句（即有`asc`又有`desc`）中,可以提高索引利用率，消除`filesort`。

MySQL 8.0 delivers support for indexes in descending order. Values in such an index are arranged in descending order, and we scan it forward. Before 8.0, when a user create a descending index, we created an ascending index and scanned it backwards. One benefit is that forward index scans are faster than backward index scans. Another benefit of a real descending index is that it enables us to use indexes instead of filesort for an `ORDER BY` clause with mixed `ASC/DESC` sort key parts. [Descending Indexes](https://dev.mysql.com/doc/refman/8.0/en/descending-indexes.html) is a frequently requested SQL feature. See for example feature request [13375](https://bugs.mysql.com/bug.php?id=13375) . See blog post by Chaithra Gopalareddy  [here](http://mysqlserverteam.com/mysql-8-0-labs-descending-indexes-in-mysql/).

[yejr]reviewing upto here

###### **分组函数 GROUPING**

MySQL 8.0 带来了`GROUPING()`分组函数，这个功能可以把`group by`子句的扩展功能（如`ROLLUP`）产生的过聚合的NULL值，通过0和1进行区分，1为NULL，[这样就可以在having子句中对过聚合的无效值进行过滤](https://www.cnblogs.com/li-peng/p/3298303.html)。

MySQL 8.0  delivers `GROUPING()`, `SQL_FEATURE T433`. The `GROUPING()` function distinguishes super-aggregate rows from regular grouped rows. `GROUP BY` extensions such as `ROLLUP` produce super-aggregate rows where the set of all values is represented by null. Using the `GROUPING()` function, you can distinguish a null representing the set of all values in a super-aggregate row from a `NULL` in a regular row. GROUPING is a frequently requested SQL feature. See feature requests [3156 ](https://bugs.mysql.com/bug.php?id=3156)and [46053](https://bugs.mysql.com/bug.php?id=46053). Thank you to *Zoe Dong* and *Shane Adams* for code contributions in feature request [46053 ](https://bugs.mysql.com/bug.php?id=46053)! See blog post by Chaithra Gopalareddy  [here](https://mysqlserverteam.com/mysql-8-0-grouping-function/).

###### **优化器建议 Optimizer Hints**

在5.7版本中我们引入了新的优化器建议的语法，借助这个新的语法，优化器建议可以被用`/*+ */`包裹起来直接放在`SELECT | INSERT | REPLACE | UPDATE | DELETE`关键字的后面。在8.0的版本中我们又加入了新的姿势。

In 5.7 we introduced a new hint syntax for [optimizer hints](https://dev.mysql.com/doc/refman/5.7/en/optimizer-hints.html). With the new syntax, hints can be specified directly after the `SELECT | INSERT | REPLACE | UPDATE | DELETE`keywords in an SQL statement, enclosed in `/*+ */` style comments. (See 5.7 blog post by Sergey Glukhov [here](https://mysqlserverteam.com/new-optimizer-hints-in-mysql/)). In MySQL 8.0 we complete the picture by fully utilizing this new style:

- 8.0版本增加了`INDEX_MERGE`和`NO_INDEX_MERGE`,允许用户在单个查询中控制是否使用索引合并特性。
- MySQL 8.0 adds hints for `INDEX_MERGE` and `NO_INDEX_MERGE`. This allows the user to control index merge behavior for an individual query without changing the optimizer switch.
- 8.0版本增加了`JOIN_FIXED_ORDER`, `JOIN_ORDER`, `JOIN_PREFIX`, 和 `JOIN_SUFFIX`，允许用户控制join表关联的顺序。
- MySQL 8.0 adds hints for `JOIN_FIXED_ORDER`, `JOIN_ORDER`, `JOIN_PREFIX`, and `JOIN_SUFFIX`. This allows the user to control table order for the join execution.
- 8.0版本增加了`SET_VAR`，该优化器建议可以设定一个只在下一条语句中生效的的系统参数。
- MySQL 8.0 adds a hint called `SET_VAR`.  The `SET_VAR` hint will set the value for a given system variable for the next statement only. Thus the value will be reset to the previous value after the statement is over. See blog post by Sergey Glukhov [here](https://mysqlserverteam.com/new-optimizer-hint-for-changing-the-session-system-variable/).

相对于之前的优化器建议和优化器特性开关参数，我们更倾向于推荐新形式的优化器建议，新形式的优化器建议可以在不侵入SQL语句（指修改语句的非注释的业务部分）的情况下，注入查询语句的很多位置。与直接修改语句的优化器建议相比，新形势的优化器建议在SQL语义上更加清晰。

We prefer the new style of optimizer hints as preferred over the old-style hints and setting of [`optimizer_switch`](https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_optimizer_switch) values. By not being inter-mingled with SQL, the new hints can be injected in many places in a query string. They also have clearer semantics in being a hint (vs directive).

### **JSON**

8.0版本追加了新的JSON函数，并可以提高在排序与分组JSON数据情况下的性能。

MySQL 8.0 adds new JSON functions and improves performance for sorting and grouping JSON values.



######  JSON path表达式中扩展的范围性语法 Extended Syntax for Ranges in JSON path expressions**

MySQL 8.0 扩展了JSON path表达式中范围性的语法，比如：`SELECT JSON_EXTRACT('[1, 2, 3, 4, 5]', '$[1 to 3]');`可以得出`[2, 3, 4]`的结果

MySQL 8.0 extends the syntax for ranges in JSON path expressions. For example `SELECT JSON_EXTRACT('[1, 2, 3, 4, 5]', '$[1 to 3]');` results in `[2, 3, 4]`. The new syntax introduced is a subset of the SQL standard syntax, described in SQL:2016, 9.39 SQL/JSON path language: syntax and semantics. See also [Bug#79052](https://bugs.mysql.com/bug.php?id=79052)reported by Roland Bouman.

###### **JSON表函数 JSON Table Functions**

MySQL 8.0 增加了可以在JSON数据上使用SQL处理工具的JSON 表函数。`JSON_TABLE()`函数可以创建JSON数据的关系型视图。可以将JSON数据估算到关系型的行列之中，用户可以对此函数返回的数据按照常规关系型数据表的方式进行SQL运算。

MySQL 8.0 adds JSON table functions which enables the use of the SQL machinery for JSON data. `JSON_TABLE()` creates a relational view of JSON  data. It maps the result of a JSON data evaluation into relational rows and columns. The user can query the result returned by the function as a regular relational table using SQL, e.g. join, project, and aggregate.

###### **JSON 聚合函数 JSON Aggregation Functions**

MySQL 8.0 增加了用于生成JSON阵列的聚合函数`JSON_ARRAYAGG()`，和用于生成JSON对象的`JSON_OBJECTAGG()`函数，令多行的JSON文档组合成JSON阵列或者JSON对象成为可能。

MySQL 8.0 adds the aggregation functions `JSON_ARRAYAGG()` to generate JSON arrays and `JSON_OBJECTAGG()` to generate JSON objects . This makes it possible to combine JSON documents in multiple rows into a JSON array or a JSON object. See blog post by Catalin Besleaga [here](http://mysqlserverteam.com/mysql-8-0-labs-json-aggregation-functions/).

###### **JSON 合并函数 JSON Merge Functions**

`JSON_MERGE_PATCH()` 函数可执行JavaScript的语法，在合并时发生重复键值对时将会优先选用第二个文档的键值对，并删除第一个文档对应的重复键值。

The `JSON_MERGE_PATCH()` function implements the semantics of JavaScript (and other scripting languages) specified by [RFC7396](https://tools.ietf.org/html/rfc7396), i.e. it removes duplicates by precedence of the second document. For example, `JSON_MERGE('{"a":1,"b":2 }','{"a":3,"c":4 }');# returns {"a":3,"b":2,"c":4}`.

`JSON_MERGE_PRESERVE()`函数与5.7版本中的` JSON_MERGE()`含义相同，都是在合并的时候保留所有值。

The `JSON_MERGE_PRESERVE()` function has the semantics of JSON_MERGE() implemented in MySQL 5.7 which preserves all values, for example  `JSON_MERGE('{"a": 1,"b":2}','{"a":3,"c":4}'); # returns {"a":[1,3],"b":2,"c":4}.`

5.7原来的`JSON_MERGE()` 函数在8.0版本中为减少merge操作的不明确，而被弃用。

The existing `JSON_MERGE()` function is deprecated in MySQL 8.0 to remove ambiguity for the merge operation. See also proposal in [Bug#81283](http://bugs.mysql.com/bug.php?id=81283) and blog post by Morgan Tocker [here](https://mysqlserverteam.com/proposal-to-change-the-behavior-of-json_merge/).

###### **JSON 美化函数 JSON Pretty Function**

8.0版本增加了可以接收JSON原生数据类型和字符串表达的JSON，并返回一行缩进的易读的JSON格式化后的的字符串。

MySQL 8.0 adds a `JSON_PRETTY()` function in MySQL. The function accepts either a JSON native data-type or string representation of JSON and returns a JSON formatted string in a human-readable way with new lines and indentation.

###### **JSON 文件大小函数 JSON Size Functions**

8.0版本增加了和指定JSON对象空间占用相关的函数，`JSON_STORAGE_SIZE()` 可以用字节为单位返回JSON某个数据类型的实际大小， `JSON_STORAGE_FREE()` 可以返回该JSON数据类型的剩余空间（包括碎片和用来适应更改后发生长度变化的预备空间）

MySQL 8.0 adds JSON functions related to space usage for a given JSON object. The `JSON_STORAGE_SIZE()` returns the actual size in bytes for a JSON datatype.  The `JSON_STORAGE_FREE()` returns the free space of a JSON binary type in bytes, including fragmentation and padding saved for inplace update.

###### **JSON 改进型的排序 JSON Improved Sorting**

8.0版本通过使用变长的排序键提升了JSON排序分组的性能。在某些场景下，Preliminary 的压测结果出现了1.2到18倍的提升。

MySQL 8.0 gives better performance for sorting/grouping JSON values by using variable length sort keys. Preliminary benchmarks shows from 1.2 to 18 times improvement in sorting, depending on use case.

###### **JSON的部分更新 JSON Partial Update**

8.0版本增加了对 `JSON_REMOVE()`, `JSON_SET()` and `JSON_REPLACE()` 函数的部分更新的支持。如果JSON文档的某部分被更新，我们会将更改的详情给到句柄。这样存储引擎和复制关系就不必写入整个JSON文档。在之前的复制环境中由于无法确保JSON文档的排列(layout)在主从上完全一致，所以在基于行的复制情况下物理文件的差异并不能用来削减传输复制信息带来的网络IO消耗。因此，8.0版本提供了在逻辑上区分差异的方法，可以在行复制的情况下传输并应用到从库上

MySQL 8.0 adds support for partial update for the `JSON_REMOVE()`, `JSON_SET()` and `JSON_REPLACE()` functions.  If only some parts of a JSON document are updated, we want to give information to the handler about what was changed, so that the storage engine and replication don’t need to write the full document. In a replicated environment, it cannot be guaranteed that the layout of a JSON document is exactly the same on the slave and the master, so the physical diffs cannot be used to reduce the network I/O for row-based replication. Thus, MySQL 8.0 provides logical diffs that row-based replication can send over the wire and reapply on the slave. See blog post by Knut Anders Hatlen [here](https://mysqlserverteam.com/partial-update-of-json-values/).

### **地理信息系统 GIS**

8.0 版本提供对地形的支持，其中包括了对空间参照系的数据源信息的支持，SRS aware spatial数据类型，空间索引，空间函数。总而言之，8.0版本可以理解地球表面的经纬度信息，而且可以在任意受支持的5000个空间参照系中计算地球上任意两点之间的距离。

MySQL 8.0 delivers geography support. This includes meta-data support for Spatial Reference System (SRS), as well as SRS aware spatial datatypes,  spatial indexes,  and spatial functions. In short, MySQL 8.0 understands latitude and longitude coordinates on the earth’s surface and can, for example, correctly calculate the distances between two points on the earths surface in any of the about 5000 supported spatial reference systems.

###### **空间参照系 Spatial Reference System (SRS)**

 `ST_SPATIAL_REFERENCE_SYSTEMS` 存在于information schema视图库中，提供了可供使用的SRS坐标系统的名称。每个SRS坐标系统都有一个SRID编号。8.0版本支持EPSG Geodetic Parameter Dataseset中的5千多个坐标系统（包括立体模和2D平面地球模型）

The `ST_SPATIAL_REFERENCE_SYSTEMS` information schema view provides information about available spatial reference systems for spatial data. This view is based on the SQL/MM (ISO/IEC 13249-3) standard. Each spatial reference system is identified by an SRID number. MySQL 8.0 ships with about 5000 SRIDs from the [EPSG Geodetic Parameter Dataset](http://www.epsg.org/EPSGhome.aspx), covering georeferenced ellipsoids and 2d projections (i.e. all 2D spatial reference systems).

###### **SRID 地理数据类型 SRID aware spatial datatypes**

空间类的数据类型可以直接从SRS坐标系统的定义中获取，例如：使用SRID 4326定义进行建表： `CREATE TABLE t1 (g GEOMETRY SRID 4326);` 。SRID是适用于地理类型的数据类型。只有同一SRID的的数据才会被插入到行中。与当前SRID数据类型的数据尝试插入时，会报错。未定义SRID编号的表将可以接受所有SRID编号的数据。

Spatial datatypes can be attributed with the spatial reference system definition, for example with SRID 4326 like this: `CREATE TABLE t1 (g GEOMETRY SRID 4326);` The SRID is here a SQL type modifier for the GEOMETRY datatype. Values inserted into a column with an SRID property must be in that SRID. Attempts to insert values with other SRIDs results in an exception condition being raised. Unmodified types, i.e., types with no SRID specification, will continue to accept all SRIDs, as before.

8.0版本增加了 `INFORMATION_SCHEMA.ST_GEOMETRY_COLUMNS` 视图，可以显示当前实例中所有地理信息的数据行及其对应的SRS名称，编号，地理类型名称。

MySQL 8.0 adds the `INFORMATION_SCHEMA.ST_GEOMETRY_COLUMNS` view as specified in SQL/MM Part 3, Sect. 19.2. This view will list all GEOMETRY columns in the MySQL instance and for each column it will list the standard `SRS_NAME` , `SRS_ID` , and `GEOMETRY_TYPE_NAME`.

###### **SRID 空间索引 SRID aware spatial indexes**

在空间数据类型上可以创建空间索引，创建空间索引的列必须非空，例如： `CREATE TABLE t1 (g GEOMETRY SRID 4326 NOT NULL, SPATIAL INDEX(g));`

Spatial indexes can be created on spatial datatypes. Columns in spatial indexes must be declared NOT NULL. For example like this: `CREATE TABLE t1 (g GEOMETRY SRID 4326 NOT NULL, SPATIAL INDEX(g));`

创建空间索引的列必须具有SRID数据标识以用于优化器使用，如果将空间索引建在没有SRID数据标识的列上，将输出waring信息。

Columns with a spatial index should have an SRID type modifier to allow the optimizer to use the index. If a spatial index is created on a column that doesn’t have an SRID type modifier, a warning is issued.

###### **SRID 空间函数 SRID aware spatial functions**

8.0 增加了诸如  `ST_Distance()` 和 `ST_Length()` 等用于判断数据的参数是否在SRS中，并计算其空间上的距离。到目前为止，`ST_Distance`和其他的空间关系型函数诸如`ST_Within`,`ST_Intersects`,`ST_Contains`,`ST_Crosses`都支持地理计算。其运算逻辑与行为参见 SQL/MM Part 3 Spatial

MySQL 8.0 extends spatial functions such as  `ST_Distance()` and `ST_Length()` to detect that its parameters are in a geographic (ellipsoidal) SRS and to compute the distance on the ellipsoid. So far, `ST_Distance` and spatial relations such as `ST_Within`, `ST_Intersects`, `ST_Contains`, `ST_Crosses`, etc.  support geographic computations. The behavior of each ST function is as defined in SQL/MM Part 3 Spatial.

### 字符集 Character Sets

8.0版本默认使用UTF8MB4作为默认字符集。相比较5.7版本，SQL性能（诸如排序UTF8MB4字符串）得到了很大的提升。UTF8MB4类型在网页编码上正占据着举足轻重的地位，将其设为默认数据类型后，将会给绝大多数的MySQL用户带来遍历。

MySQL 8.0 makes [UTF8MB4](https://dev.mysql.com/doc/refman/5.5/en/charset-unicode-utf8mb4.html) the default character set. SQL performance – such as sorting UTF8MB4 strings  – has been improved by a factor of 20 in 8.0 as compared to  5.7. UTF8MB4 is the dominating character encoding for the web, and this move will make life easier for the vast majority of MySQL users.

- 默认的字符集从`latin1`变为 `utf8mb4` ,默认排序校对规则从 `latin1_swedish_ci` 变为`utf8mb4_800_ci_ai`。
- The default character set has changed from `latin1` to `utf8mb4` and the default collation has changed from `latin1_swedish_ci` to `utf8mb4_800_ci_ai`.
-  `utf8mb4`同样也成为libmysql，服务端命令行工具,server层的默认编码
- The changes in defaults applies to libmysql and server command tools as well as the server itself.
- `utf8mb4`同样也成为MySQL测试框架的默认编码
- The changes are also reflected in MTR tests, running with new default charset.
- 排序校对规则的权重与大小写基于Unicode委员会16年公布的Unicode 9.0.0版本。
- The collation weight and case mapping are based on [Unicode 9.0.0](http://www.unicode.org/versions/Unicode9.0.0/) , announced by the Unicode committee on Jun 21, 2016.
- 在以往的MySQL版本中，`latin1`编码中的21种语言的特殊大小写和排序校对规则被引入了 `utf8mb4` 排序校对规则。例如：捷克语的排序校对规则变成了` utf8mb4_cs_800_ai_ci`。
- The 21 language specific case insensitive collations available for latin1 (MySQL legacy) have been implemented for  `utf8mb4` collations, for example the Czech collation becomes utf8mb4_cs_800_ai_ci. See complete list in [WL#9108](http://dev.mysql.com/worklog/task/?id=9108) . See blog post by Xing Zhang [here](http://mysqlserverteam.com/new-collations-in-mysql-8-0-0/) .
- 增加了对特殊语境和重音敏感的排序校对规则的支持。8.0版本支持 DUCET (Default Unicode Collation Entry Table)全部三级排序校对规则。
- Added support for case and accent sensitive collations. MySQL 8.0 supports all 3 levels of collation weight defined by DUCET (Default Unicode Collation Entry Table). See blog post by Xing Zhang [here](http://mysqlserverteam.com/mysql-8-0-1-accent-and-case-sensitive-collations-for-utf8mb4/).
-  `utf8mb4` 的 `utf8mb4_ja_0900_as_cs` 排序校验规则对日语字符支持三级权重的排序。
- Japanese `utf8mb4_ja_0900_as_cs` collation for `utf8mb4` which sorts characters by using three levels’ weight. This gives the correct sorting order for Japanese. See blog post by Xing Zhang [here](http://mysqlserverteam.com/mysql-8-0-1-japanese-collation-for-utf8mb4/).
- 对日语有额外的假名支持特性， `utf8mb4_ja_0900_as_cs_ks`中的ks表示假名区分。
- Japanese with additional kana sensitive feature, `utf8mb4_ja_0900_as_cs_ks`,  where ‘ks’ stands for ‘kana sensitive’. See blog post by Xing Zhang [here](http://mysqlserverteam.com/mysql-8-0-kana-sensitive-collation-for-japanese/).
- 把 Unicode 9.0.0之前所有排序校验规则中的不填补变成填补字符，此举有利于提升字符串的一致性和性能。例如把字符串末尾的空格按照其他字符对待。之前的排序校验规则在处理这种情况时保留字符串原样。
- Changed all new collations, from Unicode 9.0.0 forward, to be `NO PAD` instead of `PAD STRING`, ie., treat spaces at the end of a string like any other character. This is done to improve consistency and performance. Older collations are left in place.

See also blog posts by Bernt Marius Johnsen [here](http://mysqlserverteam.com/debugging-character-set-issues-by-example/), [here](http://mysqlserverteam.com/mysql-8-0-collations-migrating-from-older-collations/) and [here](http://mysqlserverteam.com/mysql-8-0-collations-the-devil-is-in-the-details/).

### **数据类型 Datatypes**

###### **二进制数据类型的Bit-wise操作 Bit-wise operations on binary data types**

8.0版本扩展了 `bit-wise`操作（如`bit-wise AND`等）的使用范围，使得其在所有 `BINARY` 数据类型上都适用。在此之前只支持整型数据，若强行在二进制数据类型上使用`Bit-wise`操作，将会隐式转换为64位的`BITINT`类型，并可能丢失若干位的数据。从8.0版本之后，bit-wise操作可以在 `BINARY` 和`BLOB`类型上使用，且不用担心精确度下降的问题。

MySQL 8.0 extends the bit-wise operations (‘bit-wise AND’, etc) to also work with `[VAR]BINARY/[TINY|MEDIUM|LONG]BLOB`. Prior to 8.0 bit-wise operations were only supported for integers. If you used bit-wise operations on binaries the arguments were implicitly cast to `BIGINT` (64 bit) before the operation, thus possibly losing bits. From 8.0 and onward bit-wise operations work for all `BINARY` and `BLOB` data types, casting arguments such that bits are not lost.

###### **IPV6操作 IPV6 manipulation**

8.0版本通过支持 `BINARY` 上的`Bit-wise`操作提升了IPv6数据的可操作性。5.6版本中引入了支持IPv6地址和16位二进制数据的互相转换的`INET6_ATON()` 和 `INET6_NTOA()` 函数。但是直到8.0之前，由于上一段中的问题我们都无法讲IPv6转换函数和`bit-wise`操作结合起来。由于 `INET6_ATON()` 可以正确的返回128bit的`VARBINARY(16)`，如果我们想要将一个IPv6地址与网关地址进行比对，现在就可以使用 `INET6_ATON(address)& INET6_ATON(network)` 操作。

MySQL 8.0 improves the usability of IPv6 manipulation supporting bit-wise operations on BINARY data types. In MySQL 5.6 we introduced  the `INET6_ATON()` and `INET6_NTOA()` functions which convert IPv6 addresses between text form like `'fe80::226:b9ff:fe77:eb17'` and `VARBINARY(16)`. However, until now we could not combine these IPv6 functions with  bit-wise operations since such operations would – wrongly – convert output to `BIGINT`. For example, if we have an IPv6 address and want to test it against a network mask, we can now use  `INET6_ATON(address)& INET6_ATON(network)` because `INET6_ATON()` correctly returns the `VARBINARY(16)`datatype (128 bits). See blog post by Catalin Besleaga [here](https://mysqlserverteam.com/mysql-8-0-storing-ipv6/).

###### **UUID 操作UUID manipulations**

8.0版本通过增加了三个新的函数（`UUID_TO_BIN()`, `BIN_TO_UUID()`, 和 `IS_UUID()`）提升了UUID的可用性。`UUID_TO_BIN()`可以将UUID格式的文本转换成`VARBINARY(16)`, `BIN_TO_UUID()`则与之相反， `IS_UUID()`用来校验UUID的有效性。将UUID以 `VARBINARY(16)` 的方式存储后，就可以使用实用的索引了。 `UUID_TO_BIN()` 函数可以原本转换后的二进制数值中的时间相关位（UUID生成时有时间关联）移到数据的开头，这样对索引来说更加友好而且可以减少在B树中的随机插入，从而减少了插入耗时。

MySQL 8.0 improves the usability of UUID manipulations by implementing three new SQL functions: `UUID_TO_BIN()`, `BIN_TO_UUID()`, and `IS_UUID()`. The first one converts from UUID formatted text to `VARBINARY(16)`, the second one from `VARBINARY(16)` to UUID formatted text, and the last one checks the validity of an UUID formatted text. The UUID stored as a `VARBINARY(16)` can be indexed using functional indexes. The functions `UUID_TO_BIN()` and `UUID_TO_BIN()` can also shuffle the time-related bits and move them at the beginning making it index friendly and avoiding the random inserts in the B-tree, this way reducing the insert time. The lack of such functionality has been mentioned as one of the [drawbacks of using UUID’s](https://www.percona.com/blog/2014/12/19/store-uuid-optimized-way/). See blog post by Catalin Besleaga [here](https://mysqlserverteam.com/mysql-8-0-uuid-support/).

### **消耗敏感的模型 Cost Model**

###### **查询优化器将会照顾到数据缓冲的状况  Query Optimizer Takes Data Buffering into Account**

8.0版本自动地根据数据是否存在于内存中而选择查询计划，在以往的版本中，消耗敏感的模型始终假设数据在磁盘上。正因为现在查询内存数据和查询硬盘数据的消耗常数不同，因此优化器会根据数据的位置选择更加优化的读取数据方式。

MySQL 8.0 chooses query plans based on knowledge about whether data resides in-memory or on-disk. This happens automatically, as seen from the end user there is no configuration involved. Historically, the MySQL cost model has assumed data to reside on spinning disks. The cost constants associated with looking up data in-memory and on-disk are now different, thus, the optimizer will choose more optimal access methods for the two cases, based on knowledge of the location of data. See blog post by Øystein Grøvlen [here](https://mysqlserverteam.com/mysql-8-0-query-optimizer-takes-data-buffering-into-account/).

###### **查询优化器的直方图 Optimizer Histograms**

8.0版本加入了直方图统计数据。用户可以根据直方图针对表中的某列（一般为非索引列）生成数据分布统计信息，这样优化器就可以利用这些信息去寻觅更加优化的查询计划。直方图最常见的使用场景就是计算字段的选择性。

MySQL 8.0 implements histogram statistics. With Histograms, the user can create statistics on the data distribution for a column in a table, typically done for non-indexed columns, which then will be used by the query optimizer in finding the optimal query plan. The primary use case for histogram statistics is for calculating the selectivity (filter effect) of predicates of the form “COLUMN operator CONSTANT”.

用以创建直方图的 `ANALYZE TABLE` 语法现已被扩展了两个新子句： `UPDATE HISTOGRAM ON column [, column] [WITH n BUCKETS]` 和`DROP HISTOGRAM ON column [, column]`。直方图的总计总数（桶）是可以选的，默认100。直方图的统计信息被存储在词典表`column_statistics`中，并可以使用`information_schema.COLUMN_STATISTICS`进行查看。由于JSON数据格式的灵活性，直方图现在以JSON对象存储。根据表的大小，`ANALYZE TABLE`命令会自动的判断是否要表进行采样，甚至会根据表中数据的分布情况和统计总量来决定创建等频或者等高的直方图。

The user creates a histogram by means of the `ANALYZE TABLE` syntax which has been extended to accept two new clauses: `UPDATE HISTOGRAM ON column [, column] [WITH n BUCKETS]` and `DROP HISTOGRAM ON column [, column]`. The number of buckets is optional, the default is 100. The histogram statistics are stored in the dictionary table “column_statistics” and accessible through the view `information_schema.COLUMN_STATISTICS`. The histogram is  stored as a JSON object due to the flexibility of the JSON datatype. `ANALYZE TABLE`  will automatically decide whether to sample the base table or not, based on table size. It will also decide whether to build a *singleton* or a *equi-height* histogram based on the data distribution and the number of buckets specified. See blog post by Erik Frøseth [here](https://mysqlserverteam.com/histogram-statistics-in-mysql/).

### **正则表达式 Regular Expressions**

与UTF8MB4的正则支持一同，8.0版本也增加了诸如 `REGEXP_INSTR()`, `REGEXP_LIKE()`, `REGEXP_REPLACE()`, 和`REGEXP_SUBSTR()`等新函数。另外，系统中还增加了用以控制正则表达式致性的 [regexp_stack_limit](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_regexp_stack_limit) (默认`8000000`比特) 和 [regexp_time_limit](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_regexp_time_limit) (默认32步) 参数。`REGEXP_REPLACE()`也是社区中受呼声比较高的特性。

MySQL 8.0 supports regular expressions for UTF8MB4 as well as new functions like `REGEXP_INSTR()`, `REGEXP_LIKE()`, `REGEXP_REPLACE()`, and `REGEXP_SUBSTR()`.  The system variables [regexp_stack_limit](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_regexp_stack_limit) (default `8000000` bytes) and [regexp_time_limit](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_regexp_time_limit) (default 32 steps) have been added to control the execution. The `REGEXP_REPLACE()`  function is one of the most requested features by the MySQL community, for example see feature request reported as [BUG #27389](http://bugs.mysql.com/bug.php?id=27389) by Hans Ginzel. See also blog posts by Martin Hansson [here](https://mysqlserverteam.com/new-regular-expression-functions-in-mysql-8-0/) and Bernt Marius Johnsen [here](https://mysqlserverteam.com/mysql-8-0-regular-expressions-and-character-properties/).

## **运维自动化特性 Dev Ops features**

开发向的运维关心数据库实例的可操作型，通常即可靠性，可用性，性能，安全，可观测性，可管理性。关于InnoDB Cluster和MGR的可靠性我们将会另起新篇单独介绍，接下来的段落将会介绍关于8.0版本针对表在其他可操作性上的改变。

Dev Ops care about operational aspects of the database, typically about reliability, availability, performance, security, observability, and manageability. High Availability comes with MySQL InnoDB Cluster and MySQL Group Replication which will be covered by a separate blog post. Here follows what 8.0 brings to the table in the other categories.

### **可靠性 Reliability**

8.0版本在整体上 增加了可靠性，原因如下：

MySQL 8.0 increases the overall reliability of MySQL because :

​       8.0版本将元信息存储与久经考验的事务性存储引擎InnoDB中。诸如用户权限表，数据字典表，现在都使用   InnoDB进行存储。

1. MySQL 8.0 stores its meta-data into InnoDB, a proven transactional storage engine.  System tables such as Users and Privileges  as well as Data Dictionary tables now reside in InnoDB.

   8.0版本消除了会导致非一致性的一处隐患。在5.7及以前的版本中，存在着服务层和引擎层两份数据字典，因而可能导致在yejr
