# 嗦一嗦 MySQL 8.0的新特性 What’s New in MySQL 8.0? (Generally Available)

原文链接：https://mysqlserverteam.com/whats-new-in-mysql-8-0-generally-available

原文作者：Geir Hoydalsvik （Oracle官方开发工程师）

翻译：张锐志

[TOC]

非常高兴的向大家宣布MySQL 8.0 GA版本发布，MySQL 8.0是一个得到全面增强且极具吸引力的新版本。不限于下面几点：

We proudly announce General Availability of MySQL 8.0. [Download now!](http://dev.mysql.com/downloads/mysql/) MySQL 8.0 is an extremely exciting new version of the world’s most popular open source database with improvements across the board. Some key enhancements include:

​       **SQL方面**：窗口函数，公共表达式，NOWAIT,SKIPLOCKED,降序索引，分组，常规表达式，字符集，基于性能损耗的优化模式，直方图

1. **SQL** Window functions, Common Table Expressions, NOWAIT and SKIP LOCKED, Descending Indexes, Grouping, Regular Expressions, Character Sets, Cost Model, and Histograms.

   **对JSON的支持**：扩充语法，新功能，增强排序，部分更新性能，基于JSON表的特性，可以使用SQL处理工具处理JSON数据。

2. **JSON** Extended syntax, new functions, improved sorting, and partial updates. With JSON table functions you can use the SQL machinery for JSON data.

   **对地理信息系统的支持**

3. **GIS** Geography support. Spatial Reference Systems (SRS), as well as SRS aware spatial datatypes,  spatial indexes,  and spatial functions.

   **可靠性**：DDL语句现在实现原子性和故障恢复（元信息数据被存在了一个基于InnoDB的单独事务性数据字典中）。

4. **Reliability** DDL statements have become atomic and crash safe, meta-data is stored in a single, transactional data dictionary. Powered by InnoDB! 

   **可观察性**：对P_S,I_S,配置参数，错误日志的记录有了极其重要的增强

5. **Observability** Significant enhancements to Performance Schema, Information Schema, Configuration Variables, and Error Logging.

   **可管理性**：远程管理，Undo表空间管理，快速DDL

6. **Manageability** Remote management, Undo tablespace management, and new instant DDL.

   **安全性**：OpenSSL的改进，新的默认验证方式，SQL角色权限，分解super权限，密码强度等等

7. **Security** OpenSSL improvements, new default authentication, SQL Roles, breaking up the super privilege, password strength, and more.

   **性能**：InnoDB在读写，带宽限制，业务热数据集中的场景上上有着举足轻重的优点，新增的资源组特性额外给用户一个在特定负载和特定硬件情况下将用户线程映射到指定的CPU上的调节选项

8. **Performance** InnoDB is significantly better at Read/Write workloads, IO bound workloads, and high contention “hot spot” workloads. Added Resource Group feature to give users an option optimize for specific workloads on specific hardware by mapping user threads to CPUs.

以上是8.0版本的部分亮点，我（原文作者）推荐您仔细阅读GA版本前几个版本的发布信息，甚至这些特性和实现方法的的项目日志。或者您可以选择直接在Github上阅读源码。

The above represents some of the highlights and I encourage you to further drill into the complete series of Milestone blog posts—[8.0.0](http://mysqlserverteam.com/the-mysql-8-0-0-milestone-release-is-available/), [8.0.1](http://mysqlserverteam.com/the-mysql-8-0-1-milestone-release-is-available/), [8.0.2](http://mysqlserverteam.com/the-mysql-8-0-2-milestone-release-is-available/), [8.0.3](https://mysqlserverteam.com/the-mysql-8-0-3-release-candidate-is-available/), and [8.0.4](https://mysqlserverteam.com/the-mysql-8-0-4-release-candidate-is-available/) —and even further down in to the individual [worklogs](http://dev.mysql.com/worklog/) with their specifications and implementation details. Or perhaps you prefer to just look at the source code at [github.com/mysql](https://github.com/mysql).

## 面向开发人员的特性  **Developer features**

MySQL 8.0应面向MySQL开发人员的需求，带来了SQL，JSON，公共表达式，地理信息系统等方面的特性，因为很多开发人员有存储EmoJi表情的需求，在新版本中UTF8MB4成为默认的字符集。除此之外，还有对Binary数据类型按位操作，和改进对IPV6和UUID函数的改进。

MySQL Developers want new features and MySQL 8.0 delivers many new and much requested features in areas such as SQL, JSON, Regular Expressions, and GIS. Developers also want to be able to store Emojis, thus UTF8MB4 is now the default character set in 8.0. Finally there are improvements in Datatypes, with bit-wise operations on BINARY datatypes and improved IPv6 and UUID functions.

### SQL

###### 窗口函数 **Window Functions**

MySQL 8.0带来了标准SQL的窗口函数功能，窗口函数与分组聚合函数相类似的是都提供了对一组行数据的统计计算。但与分组聚合函数将多行合并成一行不同是窗口函数会在结果结果集中展现每一行的聚合。

MySQL 8.0 delivers SQL window functions.   Similar to grouped aggregate functions, window functions perform some calculation on a set of rows, e.g. `COUNT` or `SUM`. But where a grouped aggregate collapses this set of rows into a single row, a window function will perform the aggregation for each row in the result set.

窗口函数有两种使用方式，首先是常规的SQL聚合功能函数和特殊的窗口函数。常规的聚合功能函数如：`COUNT`,`SUM`等函数。而窗口函数专有的则是`RANK`, `DENSE_RANK`, `PERCENT_RANK`, `CUME_DIST`, `NTILE`, `ROW_NUMBER`, `FIRST_VALUE`, `LAST_VALUE`, `NTH_VALUE`, `LEAD `and `LAG`等函数

Window functions come in two flavors: SQL aggregate functions used as window functions and specialized window functions. This is the set of aggregate functions in MySQL that support windowing: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `BIT_OR`, `BIT_AND`, `BIT_XOR`, `STDDEV_POP` (and its synonyms `STD`, `STDDEV`), `STDDEV_SAMP`, `VAR_POP` (and its synonym `VARIANCE`) and `VAR_SAMP`. The set of specialized window functions are: `RANK`, `DENSE_RANK`, `PERCENT_RANK`, `CUME_DIST`, `NTILE`, `ROW_NUMBER`, `FIRST_VALUE`, `LAST_VALUE`, `NTH_VALUE`, `LEAD `and `LAG`

对窗口函数的支持上，是用户呼声比较频繁。窗口函数早在SQL2003规范中就成为了标准SQL的一部分。

Support for window functions (a.k.a. analytic functions) is a frequent user request. Window functions have long been part of standard SQL (SQL 2003). See blog post by Dag Wanvik [here](http://mysqlserverteam.com/mysql-8-0-2-introducing-window-functions/) as well as blog post by Guilhem Bichot [here](https://mysqlserverteam.com/row-numbering-ranking-how-to-use-less-user-variables-in-mysql-queries/).

###### 公用表达式

###### **Common Table Expression**

MySQL 8.0 带来了支持递归的公用表达式的功能。非递归的公用表达式由于允许由form子句派生的临时表的原因可以被多次引用，因而被解释为改进型的派生表（from子句中的临时表）。而递归的公用表达式则由一组原始住居，经过处理后得到新的一组数据，再被带入处理得到更多的新数据，循环往复直到再也无法产生更多新数据为止。公用表达式也是一个用户呼声频繁的SQL功能。

MySQL 8.0 delivers [Recursive] Common Table Expressions (CTEs).  Non-recursive CTEs can be explained as “improved derived tables” as it allow the derived table to be referenced more than once. A recursive CTE is a set of rows which is built iteratively: from an initial set of rows, a process derives new rows, which grow the set, and those new rows are fed into the process again, producing more rows, and so on, until the process produces no more rows. CTE is a commonly requested SQL feature, see for example feature request [16244 ](https://bugs.mysql.com/bug.php?id=16244)and [32174 ](https://bugs.mysql.com/bug.php?id=32174). See blog posts by Guilhem Bichot [here](http://mysqlserverteam.com/mysql-8-0-labs-recursive-common-table-expressions-in-mysql-ctes/), [here](http://mysqlserverteam.com/mysql-8-0-labs-recursive-common-table-expressions-in-mysql-ctes-part-two-how-to-generate-series/), [here](http://mysqlserverteam.com/mysql-8-0-labs-recursive-common-table-expressions-in-mysql-ctes-part-three-hierarchies/), and [here](http://mysqlserverteam.com/mysql-8-0-1-recursive-common-table-expressions-in-mysql-ctes-part-four-depth-first-or-breadth-first-traversal-transitive-closure-cycle-avoidance/).

###### 立即报错或者跳过锁持有的行

###### NOWAIT and SKIP LOCKED**

MySQL 8.0 给SQL的上锁子句带来了`NOWAIT`和`SKIP LOCKED`两个额外的可选项。在原来的版本中，当行数据被`UPDATE`或者`SELECT ... FOR UPDATE`语句上锁后，其他的事务需要等待锁释放才能访问这行数据。但在某些场景下，有着要马上获取反馈的（不等待锁）需求。使用`NOWAIT`参数后如果请求的数据中包括了被锁住的行，将马上会收到查询失败的报错信息。使用`SKIP LOCKED`参数后，返回的数据将会跳过被锁住的行。

MySQL 8.0 delivers `NOWAIT` and `SKIP LOCKED` alternatives in the SQL locking clause. Normally, when a row is locked due to an `UPDATE` or a `SELECT ... FOR UPDATE`, any other transaction will have to wait to access that locked row. In some use cases there is a need to either return immediately if a row is locked or ignore locked rows. A locking clause using `NOWAIT` will never wait to acquire a row lock. Instead, the query will fail with an error. A locking clause using `SKIP LOCKED` will never wait to acquire a row lock on the listed tables. Instead, the locked rows are skipped and not read at all. NOWAIT and SKIP LOCKED are frequently requested SQL features. See for example feature request [49763 ](https://bugs.mysql.com/bug.php?id=49763). We also want to say thank you to *Kyle Oppenheim* for his code contribution! See blog post by Martin Hansson [here](https://mysqlserverteam.com/mysql-8-0-1-using-skip-locked-and-nowait-to-handle-hot-rows/).

###### 降序索引

###### Descending Indexes**

MySQL 8.0 带来了对降序索引的支持。在 8.0降序索引中，数据被倒序组织，正向查找。而在之前的版本中，虽然支持创建降序排列的索引，但其实现方式是通过创建常见的正序索引，然后进行反向查找来实现的。一方面来说，正序查找要比逆序查找更快，另一方面来说，真正的降序索引在复合的order by语句（即有`asc`又有`desc`）中,可以提高索引利用率，减少`filesort`。

MySQL 8.0 delivers support for indexes in descending order. Values in such an index are arranged in descending order, and we scan it forward. Before 8.0, when a user create a descending index, we created an ascending index and scanned it backwards. One benefit is that forward index scans are faster than backward index scans. Another benefit of a real descending index is that it enables us to use indexes instead of filesort for an `ORDER BY` clause with mixed `ASC/DESC` sort key parts. [Descending Indexes](https://dev.mysql.com/doc/refman/8.0/en/descending-indexes.html) is a frequently requested SQL feature. See for example feature request [13375](https://bugs.mysql.com/bug.php?id=13375) . See blog post by Chaithra Gopalareddy  [here](http://mysqlserverteam.com/mysql-8-0-labs-descending-indexes-in-mysql/).

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

   8.0版本消除了会导致非一致性的一处隐患。在5.7及以前的版本中，存在着服务层和引擎层两份数据字典，因而可能导致在故障情况下的数据字典间的同步失败。在8.0版本中，只有一份数据字典。

2. MySQL 8.0 eliminates one source of potential inconsistency.  In 5.7 and earlier versions there are essentially two data dictionaries, one for the Server layer and one for the InnoDB layer, and these can get out of sync in some crashing scenarios.   In 8.0 there is only one data dictionary.

   8.0版本实现了原子化，无惧宕机的DDL。根据这个特性，DDL语句要么被全部执行，要么全部未执行。对于复制环境来说这是至关重要的，否则会导致主从之间因为表结构不一致，数据漂移的情况。

3. MySQL 8.0 ensures atomic, crash safe DDL. With this the user is guaranteed that any DDL statement will either be executed fully or not at all. This is particularly important in a replicated environment, otherwise there can be scenarios where masters and slaves (nodes) get out of sync, causing data-drift.

基于新的事务型数据字典，可靠性得到了提高。

This work is done in the context of the new, transactional data dictionary. See blog posts by Staale Deraas [here](https://mysqlserverteam.com/atomic-ddl-in-mysql-8-0/) and [here](https://mysqlserverteam.com/mysql-8-0-data-dictionary-status-in-the-8-0-0-dmr/).

### **可观测性 Observability**

###### **提高信息视图库的性能 Information Schema (speed up)**

8.0版本重新实现了信息视图库，在新的实现中，信息视图库的表都是基于InnoDB存储的数据字典表的简单视图。这比之前有了百倍的性能提升。让信息视图库可以更加实用性的被外部工具引用。

MySQL 8.0 reimplements Information Schema. In the new implementation the Information Schema tables are simple views on data dictionary tables stored in InnoDB. This is by far more efficient than the old implementation with up to 100 times speedup. This makes Information Schema practically usable by external tooling. See blog posts by  Gopal Shankar [here](http://mysqlserverteam.com/mysql-8-0-improvements-to-information_schema/) and [here](http://mysqlserverteam.com/mysql-8-0-scaling-and-performance-of-information_schema/) , and the  blog post by Ståle Deraas [here](https://mysqlserverteam.com/further-improvements-on-information_schema-in-mysql-8-0-3/).

###### **提高性能信息库的速度 Performance Schema (speed up)**

8.0版本通过在性能信息库上增加了100多个索引完成了其查询性能的提升。这些索引是预设，且无法被删除，修改，增加。相对于在单独的数据结构上进行便利，其索引是通过在既存数据表上过滤扫描来实现的。不需要管理B树，或者散列表的重建，更新及其他操作。性能信息库的索引从行为上更像散列索引：1，可以快速返回数据，2，不支持排序操作（推到服务层处理）。根据查询，索引会避免全表扫描的需求，而且会返回一个相当精巧的数据集。对`show indexes`来说，性能信息库的索引是不可见的，但是会出现在`explain`的结果中和其有关的部分。

MySQL 8.0 speeds up performance schema queries by adding more than 100 indexes on performance schema tables.  The indexes on performance schema tables are predefined. They cannot be deleted,added or altered. A performance schema index is implemented as a filtered scan across the existing table data, rather than a traversal through a separate data structure. There are no B-trees or hash tables to be constructed, updated or otherwise managed. Performance Schema tables indexes behave like hash indexes in that a) they quickly retrieve the desired rows, and b) do not provide row ordering, leaving the server to sort the result set if necessary. However, depending on the query, indexes obviate the need for a full table scan and will return a considerably smaller result set. Performance schema indexes are visible with `SHOW INDEXES` and are represented in the `EXPLAIN` output for queries that reference indexed columns. See [comment](http://mysqlserverteam.com/using-sys-session-as-an-alternative-to-show-processlist/#comment-11037) from Simon Mudd. See blog post by Marc Alff [here](https://mysqlserverteam.com/mysql-8-0-performance-schema-now-with-indexes/).

###### **参数配置 Configuration Variables**

8.0版本增加了关于配置参数的有用信息，比如变量名，最大最小值，当前值的来源，更改用户，更改时间等等。可以在一个新增的性能信息表`variables_info`表中进行查询。

MySQL 8.0 adds useful information about configuration variables, such as the variable name, *min/max* values, *where* the current value came from,  *who* made the change and *when* it was made. This information is found in a new performance schema table called  `variables_info`. See blog post by  Satish Bharathy [here](https://mysqlserverteam.com/mysql-8-0-persisting-configuration-variables/).

###### **客户端错误报告信息统计 Client Error Reporting – Message Counts**

8.0版本使得查看服务端曝出的客户端错误信息汇总统计成为可能。用户可以在五个不同的表中查看统计信息：`Global count`, `summary per thread`, `summary per user`, `summary per host`, 和`summary per account`。

用户可以查看单个错误信息的次数，被`SQL exception`句柄处理过的数量，第一次发生的时间戳，最近一次的时间戳。给予用户适当的权限后，用户既可以用`select`从中查询，也可以使用`truncate`进行重置统计数据。

MySQL 8.0 makes it possible to look at aggregated counts of client  [error messages](http://dev.mysql.com/doc/refman/5.7/en/error-messages-server.html)reported by the server.The user can look at statistics from 5 different tables: Global count, summary per thread, summary per user, summary per host, or summary per account. For each error message the user can see the number of errors raised, the number of errors handled by the SQL exception handler, “first seen” timestamp, and “last seen” timestamp. Given the right privileges the user can either `SELECT` from these tables or `TRUNCATE` to reset statistics. See blog post by Mayank Prasad [here](https://mysqlserverteam.com/mysql-8-0-performance-schema-instrumentation-of-server-errors/).

###### **语句延迟直方图 Statement Latency  Histograms**

为了更高的观察查询相应时间，8.0版本在性能信息库中提供了语句延迟的直方图，同时会从直方图中计算95%，99%，9999%比例的信息。这些百分比用来作为服务质量的指示器。

MySQL 8.0 provides performance schema histograms of statements latency, for the purpose of better visibility of query response times. This work also computes “P95”, “P99” and “P999” percentiles from collected histograms. These percentiles can be used as indicators of quality of service. See blog post by Frédéric Descamps [here](http://lefred.be/content/mysql-8-0-statements-latency-histograms/).

###### **图形化数据依赖关系锁 Data Locking Dependencies Graph**

8.0版本在性能信息库中增加了数据锁生产者身份。当事务A锁住行R时，事务B正在等待一行被A锁住的行。新增的生产者身份将会那些数据被锁住（本例中为行R），谁拥有锁（本例中为事务A），谁在等待被锁住的事务（本例中为事务B）

MySQL 8.0 instruments data locks in the performance schema. When transaction A is locking row R, and transaction B is waiting on this very same row, B is effectively blocked by A. The added instrumentation exposes which data is locked (R), who owns the lock (A), and who is waiting for the data (B). See blog post by Frédéric Descamps [here](http://lefred.be/content/mysql-8-0-data-locking-visibility/).

###### **查询样例摘要 Digest Query Sample**

8.0版本为了捕获完成的查询样例和此查询案例的一些关键信息，针对性能信息库中的` events_statements_summary_by_digest`表做了一些改动。为了捕获一个真实的查询，并让用户进行`explain`，并获取查询计划，现增加了一列`QUERY_SAMPLE_TEXT` 。为了捕获查询样例时间戳，增加了一列`QUERY_SAMPLE_SEEN` 。为了捕获查询执行时间，增加了一列 `QUERY_SAMPLE_TIMER_WAIT` 。同时`FIRST_SEEN` 和 `LAST_SEEN` 列被修改为可以使用带小数的秒。

MySQL 8.0 makes some changes to the [events_statements_summary_by_digest](https://dev.mysql.com/doc/refman/8.0/en/statement-summary-tables.html)performance schema table to capture a full example query and some key information about this query example. The column `QUERY_SAMPLE_TEXT` is added to capture a query sample so that users can run EXPLAIN on a real query and to get a query plan. The column `QUERY_SAMPLE_SEEN` is added  to capture the query sample timestamp. The column `QUERY_SAMPLE_TIMER_WAIT` is added to capture the query sample execution time. The columns `FIRST_SEEN` and `LAST_SEEN`  have been modified to use fractional seconds. See blog post by Frédéric Descamps [here](http://lefred.be/content/mysql-8-0-digest-query-samples-in-performance_schema/).

###### **生产者的元信息 Meta-data about Instruments**

8.0版本在性能信息库的  [setup_instruments](https://dev.mysql.com/doc/refman/8.0/en/setup-instruments-table.html)表上增加了诸如属性，易变的，文档等元信息。这些只读信息作为生产者的在线文档被用户或者工具查阅。

MySQL 8.0  adds meta-data such as *properties*, *volatility*, and *documentation* to the performance schema table  [setup_instruments](https://dev.mysql.com/doc/refman/8.0/en/setup-instruments-table.html). This read only meta-data act as online documentation for instruments, to be looked at by users or tools. See blog post by Frédéric Descamps [here](http://lefred.be/content/mysql-8-0-meta-data-added-to-performance_schemas-instruments/).

###### 错误记录 Error Logging**

8.0版本 带来了对错误日志的重要的改革。从软件架构的角度来说，错误日志成为了新的服务架构的一部分。这意味着高级用户可以根据需要写出自己的错误日志实现。虽然大多数用户并不想这么做，但是或许会在如何写，写在何处上要求有些变通的空间。因此8.0版本为用户提供了sinks和filters来实现。8.0版本实行了一个过滤服务（API），和一个默认的过滤服务实现（组件）。这里的过滤器是指抑制某些特定错误信息的数据和或给定错误信息的某部分的输出。8.0版本实行了一个日志撰写（API），和一个默认的日志撰写服务实现（组件）。日志撰写器可以接收日志信息，并将其写入到日志中，这里的日志可以指典型的文件日志，syslog日志，或者JSON日志。

MySQL 8.0 delivers a major overhaul of the MySQL [error log](https://dev.mysql.com/doc/refman/8.0/en/error-log.html). From a software architecture perspective the error log is made a component in the new  service infrastructure. This means that advanced users can write their own error log implementation if desired. Most users will not want to write their own error log implementation but still want some flexibility in what to write and where to write it.  Hence, 8.0 offers users facilities to add *sinks (where)* and *filters (what)*.  MySQL 8.0 implements a filtering service (API) and a default filtering service implementation (component). Filtering here means to suppress certain log messages (selection) and/or fields within a given log message (projection). MySQL 8.0 implements a log writer service (API) and a default log writer service implementation (component).  Log writers accept a log event and write it to a log. This log can be a classic file, syslog, EventLog and a new JSON log writer.

不做任何设置默认的话，8.0版本带来了开箱即用的错误日志改进。比如：

By default, without any configuration, MySQL 8.0 delivers many out-of-the-box error log improvements such as:

- **错误编码：** 编码格式现在为MY开头的10000系列数据。比如： “MY-10001”。错误编号在GA版本中将不会变化，但是其代表的含义可能在之后的维护性版本发布中做一些改变。
- **Error numbering:** The format is a number in the 10000 series preceded by “MY-“, for example “MY-10001”. Error numbers will be stable in a GA release, but the corresponding error texts are allowed to change (i.e. improve) in maintenance releases.
- **系统信息：**系统性的信息[System]替换了之前的 [Error]*（指之前错误日志是系统性相关，但是前缀为[Error]的错误信息）*,增加到错误日志中。
- **System messages:** System messages are written to the error log as [System] instead of [Error], [Warning], [Note]. [System] and [Error] messages are printed regardless of verbosity and cannot be suppressed.  [System] messages are only used in a few places, mainly associated with major state transitions such as starting or stopping the server.
- **错误日志详细度减弱：** 默认的错误日志详细信息级别` log_error_verbosity` 从3(输出)改成了2（输出警告级别及以上）
- **Reduced verbosity:** The default of [log_error_verbosity](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_log_error_verbosity) changes from 3 (Notes) to 2 (Warning). This makes MySQL 8.0 error log less verbose by default.
- **信息源部分：**每个信息前面都加了[Server], [InnoDB], [Replic] 三个其中之一的注释，以显示信息是从哪个子系统中输出的。
- **Source Component:** Each message is annotated with one of three values [Server], [InnoDB], [Replic] showing which sub-system the message is coming from.

8.0GA版本错误日志中的启动信息：

This is what is written to the error log  in 8.0 GA after startup :

| 1234 | 2018-03-08T10:14:29.289863Z 0 [System][MY-010116] [Server] /usr/sbin/mysqld (mysqld 8.0.5) starting as process 8063                                                                                                                          2018-03-08T10:14:29.745356Z 0 [Warning][MY-010068] [Server] CA certificate ca.pem is self signed.                  2018-03-08T10:14:29.765159Z 0 [System][MY-010931] [Server] /usr/sbin/mysqld: ready for connections. Version: '8.0.5'  socket: '/tmp/mysql.sock'  port: 3306  Source distribution.                                                                                                                                    2018-03-08T10:16:51.343979Z 0 [System][MY-010910] [Server] /usr/sbin/mysqld: Shutdown complete (mysqld 8.0.5)  Source distribution. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

新引入的错误编码方式允许 MySQL在需要的情况下在以后的维护性版本发布中，不改变错误编号的情况下改进错误详细信息。同时错误编号还可以用于在客制化中作为过滤/屏蔽错误信息的实施基础。

The introduction of error numbering in the error log allows MySQL to improve an error text in upcoming maintenance releases (if needed) while keeping the error number (ID) unchanged.   Error numbers also act as the basis for filtering/suppression  and internationalization/localization.

### **可管理性 Manageability**

###### **不可见索引 INVISIBLE Indexes**

8.0版本将索引的可见性管理变为可能。一个不可见索引在优化器指定查询执行计划时不会被纳入考虑。但是这个索引仍会在后台维护，因此让其变得可见比删除再增加索引代价更小。设计不可见索引的目的时为了帮助DBA来判断索引能否被删除。如果你假设索引已经不会被用到，可以先将其设为不可见，然后观察查询性能，如果相关的查询性能没有下降的话，最后可以将其删除。这个功能也是万众期盼的。

MySQL 8.0 adds the capability of toggling the visibility of an index (visible/invisible). An invisible index is not considered by the optimizer when it makes the query execution plan. However, the index is still maintained in the background so it is cheap to make it visible again. The purpose of this is for a DBA / DevOp to determine whether an index can be dropped or not. If you suspect an index of not being used you first make it invisible, then monitor query performance, and finally remove the index if no query slow down is experienced. This feature has been asked for by many users, for example through [Bug#70299](https://bugs.mysql.com/bug.php?id=70299). See blog post by Martin Hansson [here](https://mysqlserverteam.com/mysql-8-0-invisible-indexes/).

###### **灵活的Undo表空间管理 Flexible Undo Tablespace Management**

8.0版本让用户可以全权管理Undo表空间，比如表空间的数量，存放位置，每个undo表空间有多少回滚段。

MySQL 8.0 gives the user full control over Undo tablespaces, i.e. *how many*tablespaces, *where* are they placed, and *how many rollback segments* in each.

​       *Undo日志不在存放于系统表空间中：*在版本升级的过程中，Undo日志被从系统表空间中分离出来，并放到Undo表空间中。这为现存的非独立Undo表空间的升级留了后路。

1. *No more Undo log in the System tablespace.* Undo log is migrated out of the System tablespace and into Undo tablespaces during upgrade. This gives an upgrade path for existing 5.7 installation using the system tablespace for undo logs.

   *Undo表空间可以单独管理*：比如放到更快的磁盘存储上。

2. *Undo tablespaces can be managed separately from the System tablespace.*For example, Undo tablespaces can be put on fast storage.

   *在线Undo表空间回收：*为了Undo表空间清理的需要，生成了了两个小的Undo表空间，这样可以让InnoDB在一个活跃一个清理的情况下在线收缩Undo表空间。

3. *Reclaim space taken by unusually large transactions (online).*  A minimum of two Undo tablespaces are created to allow for tablespace truncation. This allows InnoDB to shrink the undo tablespace because one Undo tablespace can be active while the other is truncated.

   *增多回滚段，减少争用：*现在用户可以选择使用最多127个Undo表空间，每个表空间都有最多可达128个回滚段。更多的回滚段可以让并行的事务更可能地为其undo日志使用独立的回滚段，这样可以减少对同个资源的争用。

4. *More rollback segments results in less contention.* The  user might choose to have up to 127 Undo tablespaces, each one having up to 128 rollback segments. More rollback segments mean that concurrent transactions are more likely to use separate rollback segments for their undo logs which results in less contention for the same resources.

See blog post by Kevin Lewis [here](https://mysqlserverteam.com/mysql-8-0-2-more-flexible-undo-tablespace-management/).

###### **将全局参数设置持久化 SET PERSIST for global variables**

在正常的情况下，全局的动态的参数可以在线更改，但是实例重启后，这些没有写入配置文件或者与配置文件冲突的参数设定值有可能会丢失。8.0版本中可以将全局的动态参数的更改持久化。

MySQL 8.0 makes it possible to persist global, dynamic server variables. Many server variables are both GLOBAL and DYNAMIC and can be reconfigured while the server is running. For example: `SET GLOBAL sql_mode='STRICT_TRANS_TABLES'; `However, such settings are lost upon a server restart.

这就让 `SET PERSIST sql_mode='STRICT_TRANS_TABLES'; `的写法成为可能。这样就可以在重启后，参数设定值仍会留存。这个功能的使用场景很多，但是最重要的是给出了一个在更改配置文件不方便或者根本无法做到的情况下的选择。比如某些托管的场景下，你根本没有文件系统的访问权限，仅有连入数据库服务的权限。和 `SET GLOBAL` 相同，`SET PERSIST`也需要`super`权限。

This work makes it possible to write `SET PERSIST sql_mode='STRICT_TRANS_TABLES'; `The effect is that the setting will survive a server restart. There are many usage scenarios for this functionality but most importantly it gives a way to manage server settings when editing the configuration files is inconvenient or not an option. For example in some hosted environments you don’t have file system access, all that you have is the ability to connect to one or more servers. As for `SET GLOBAL` you need the super privilege for `SET PERSIST`.

除此之外还有 `RESET PERSIST` 命令，可以将之前使用`SET PERSIST`命令更改后参数值的持久化特性取消掉，让这个参数值和 `SET GLOBAL`设置的一样，下次启动可能会丢失。

There is also the `RESET PERSIST` command. The `RESET PERSIST` command has the semantic of removing the configuration variable from the persist configuration, thus converting it to have similar behavior as `SET GLOBAL`.

8.0版本也允许对大部分当前启动环境下的只读参数进行 `SET PERSIST` 更改，在下次启动时会生效。当然了部分只读参数是无法被设置更改的。

MySQL 8.0 allows `SET PERSIST` to set most read-only variables as well, the new values will here take effect at the next server restart.  Note that a small subset of read-only variables are left intentionally not settable. See blog post by  Satish Bharathy [here](https://mysqlserverteam.com/mysql-8-0-persisting-configuration-variables/).

###### **远程管理 Remote Management**

8.0版本增加了` RESTART`命令。其用途是用来允许通过SQL连接来允许远程管理MySQL实例。比如通过 `SET PERSIST` 命令更改了只读参数后，可以用这个命令来重启MySQL实例。当然，需要`shutdown`权限（译者注：同时mysqld进程需要被supervisor进程管理才可以。）。

MySQL 8.0 implements an SQL RESTART command. The purpose is to enable remote management of a MySQL server over an SQL connection, for example to set a non-dynamic configuration variable by `SET PERSIST` followed by a `RESTART`.  See blog post [MySQL 8.0: changing configuration easily and cloud friendly !](http://lefred.be/content/mysql-8-0-changing-configuration-easily-and-cloud-friendly/)  by Frédéric Descamps.

###### **重命名表空间 Rename Tablespace (SQL DDL)**

8.0版本增加了 `ALTER TABLESPACE s1 RENAME TO s2;`功能，通用表空间或者共享表空间都可以被用户创建，更改，删除。

MySQL 8.0 implements `ALTER TABLESPACE s1 RENAME TO s2;` A shared/general tablespace is a user-visible entity which users can CREATE, ALTER, and DROP. See also [Bug#26949](https://bugs.mysql.com/bug.php?id=26949), [Bug#32497](https://bugs.mysql.com/bug.php?id=32497), and [Bug#58006](https://bugs.mysql.com/bug.php?id=58006).

###### **列重命名 Rename Column (SQL DDL)**

8.0版本支持 `ALTER TABLE ... RENAME COLUMN old_name TO new_name;`，这相对于现有的` ALTER TABLE <table_name> CHANGE … `语法（需要注明当前列所有属性）来说是个改进。应用程序可能由于某些原因并不能获取所该列的所有信息，因此，这是之前语法的弊端。同时之前的语法也可能会偶然导致数据丢失（rename应该是在线DDL，不需要重建表）

MySQL 8.0 implements `ALTER TABLE ... RENAME COLUMN old_name TO new_name;`This is an improvement over existing syntax ALTER TABLE <table_name> CHANGE … which requires re-specification of all the attributes of the column. The old/existing syntax has the disadvantage that all the column information might not be available to the application trying to do the rename. There is also a risk of accidental data type change in the old/existing syntax which might result in data loss.

### **安全特性 Security features**

###### **新的默认验证插件 New Default Authentication Plugin**

8.0版本将默认的验证插件从`mysql_native_password`改成了`caching_sha2_password`。与此对应的，libmysqlclient也会采用`caching_sha2_password `验证机制。新的`caching_sha2_password`结合了更高的安全特性（SHA2算法）和高性能（缓存）。我们目前建议所有用户在网络通信上使用TLS/SSL。

MySQL 8.0 changes the default authentication plugin from [mysql_native_password](https://dev.mysql.com/doc/refman/8.0/en/native-pluggable-authentication.html)to [caching_sha2_password](https://dev.mysql.com/doc/refman/8.0/en/caching-sha2-pluggable-authentication.html). Correspondingly, libmysqlclient will  use caching_sha2_password as the default authentication mechanism, too. The new  caching_sha2_password  combines better security (SHA2 algorithm) with high performance (caching). The general direction is that we recommend all users to use TLS/SSL for all  their network communication. See blog post by Harin Vadodaria [here](https://mysqlserverteam.com/mysql-8-0-4-new-default-authentication-plugin-caching_sha2_password/).

###### **社区版本默认使用OpenSSL OpenSSL by Default in Community Edition**

8.0版本正在把OpenSSL作为企业版本和社区版本的统一默认TLS/SSL库。之前社区版本使用的是YaSSL。支持OpenSSL也是社区呼声比较多的请求了。

MySQL 8.0 is unifying on [OpenSSL](https://en.wikipedia.org/wiki/OpenSSL) as the default TLS/SSL library for both MySQL Enterprise Edition and MySQL Community Edition.  Previously, MySQL Community Edition used [YaSSL](https://en.wikipedia.org/wiki/WolfSSL). Supporting OpenSSL in the MySQL Community Edition has been one of the most frequently requested features. See blog post by Frédéric Descamps [here](https://mysqlserverteam.com/mysql-8-0-4-openssl-and-mysql-community-edition/).

###### **动态链接的OpenSSL OpenSSL is Dynamically Linked**

8.0版本动态的和OpenSSL链接在一起。从MySQL软件源的使用者角度来看，现在MySQL包依赖于系统提供的OpenSSL文件。动态的链接之后，OpenSSL的更新就不需要要求MySQL也一同更新或者打补丁。

MySQL 8.0 is linked dynamically with OpenSSL. Seen from the [MySQL Repository](https://dev.mysql.com/downloads/repo/)users perspective , the MySQL packages depends on the OpenSSL files provided by the Linux system at hand. By dynamically linking, OpenSSL updates can be applied upon availability without requiring a MySQL upgrade or patch. See blog post by Frédéric Descamps [here](https://mysqlserverteam.com/mysql-8-0-4-openssl-and-mysql-community-edition/).

###### **对Undo和Redo日志的加密 Encryption of Undo and Redo log**

8.0版本加入了对Undo和Redo日志的静态加密。在5.7版本中，我们引入了对使用了独立表空间的InnoDB表的加密。其可为物理的表空间数据文件提供静态加密。在8.0版本中我们将这个贴行扩展到了Undo和Redo日志上。

MySQL 8.0 implements data-at-rest encryption of UNDO and REDO logs. In 5.7 we introduced [Tablespace Encryption](https://dev.mysql.com/doc/refman/5.7/en/innodb-tablespace-encryption.html) for InnoDB tables stored in file-per-table tablespaces. This feature provides at-rest encryption for physical tablespace data files. In 8.0 we extend this to include UNDO and REDO logs.  See documentation [here](https://dev.mysql.com/doc/refman/8.0/en/innodb-tablespace-encryption.html).

###### **SQL 角色SQL roles**

8.0版本引入了SQL角色。角色是一组权限的集合。其目的是为了简化用户权限管理系统。可以角色赋给用户的身上，给角色授权，创建角色，删除角色，定义哪些角色对于当前会话是合适的。

MySQL 8.0 implements SQL Roles. A role is a named collection of privileges. The purpose is to simplify the user access right management. One can grant roles to users, grant privileges to roles, create roles, drop roles, and decide what roles are applicable during a session. See blog post by Frédéric Descamps [here](http://lefred.be/content/mysql-8-0-listing-roles/).

###### **允许为公用角色授权或收回权限  Allow grants and revokes for PUBLIC**

8.0版本引入了可配置的参数 [`mandatory-roles`](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_mandatory_roles)，用于自动给新创建的用户加上角色。授予给所有用户指定的角色的权限不可以被再次分发。但是这些角色除非被设置为默认角色，否则还是需要进行激活操作。当然也可以把新引入的 `activate-all-roles-on-login`参数设为`ON`，这样在用户验证通过连接进来后，所有授权角色都会自动激活。

MySQL 8.0 introduces the configuration variable [`mandatory-roles`](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_mandatory_roles) which can be used for automatic assignment and granting of *default roles* when new users are created. Example: `role1@%,role2,role3,role4@localhost`.  All the specified roles are always considered granted to every user and they can’t be revoked. These roles still require activation unless they are made into default roles. When the new server configuration variable [`activate-all-roles-on-login`](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_activate_all_roles_on_login) is set to “ON”, all granted roles are always activated after the user has authenticated.

###### **分解super权限 Breaking up the super privileges**

8.0版本定义了在很多方面上一批新粒度的权限用以代替之前版本使用的`SUPER`权限。其本意是用于限制用户仅获得和自己工作相关的权限。比如 [BINLOG_ADMIN](https://dev.mysql.com/doc/refman/8.0/en/privileges-provided.html#priv_binlog-admin), [CONNECTION_ADMIN](https://dev.mysql.com/doc/refman/8.0/en/privileges-provided.html#priv_connection-admin), and [ROLE_ADMIN](https://dev.mysql.com/doc/refman/8.0/en/privileges-provided.html#priv_role-admin).

MySQL 8.0  defines a set of new granular privileges for various aspects of what SUPER is used for in previous releases. The purpose is to limit user access rights to what is needed for the job at hand and nothing more. For example [BINLOG_ADMIN](https://dev.mysql.com/doc/refman/8.0/en/privileges-provided.html#priv_binlog-admin), [CONNECTION_ADMIN](https://dev.mysql.com/doc/refman/8.0/en/privileges-provided.html#priv_connection-admin), and [ROLE_ADMIN](https://dev.mysql.com/doc/refman/8.0/en/privileges-provided.html#priv_role-admin).

###### **用于管理XA事务的授权模型 Authorization model to manage XA-transactions**

8.0版本引入了一个新的系统权限 `XA_RECOVER_ADMIN` ，用于控制执行 `XA RECOVER`语句的权限。所有尝试执行 `XA RECOVER` 语句的非授权用户会引起报错。

MySQL 8.0 introduces a new system privilege `XA_RECOVER_ADMIN` which controls the capability to execute the statement `XA RECOVER`. An attempt to do `XA RECOVER` by a user who wasn’t granted the new system privilege `XA_RECOVER_ADMIN` will cause an error.

###### **密码轮换策略 Password rotation policy**

8.0版本引入了对密码重新使用的限制，既可以在全局层级也可以在单独的用户等级上配置。过往的历史密码由于安全的原因（会泄露密习惯，或者词组）会被加密保存。密码轮换策略对于其他策略来说是叠加的。可以和现有的机制（如，密码过期和密码安全策略等）共存。

MySQL 8.0 introduces restrictions on password reuse. Restrictions can be configured at global level as well as individual user level. Password history is kept secure because it may give clues about habits or patterns used by individual users when they change their password. The *password rotation policy* comes in addition to other, existing mechanisms such as the *password expiration policy* and *allowed password policy*. See [Password Management](https://dev.mysql.com/doc/refman/8.0/en/password-management.html).

###### **减缓对用户密码的暴力破解 Slow down brute force attacks on user passwords**

8.0版本引入了在连续的错误登陆尝试后的等待验证过程。其设计目的用于减缓哦对用户密码的暴力破解。密码连续错误次数和连续错误之后的等待时间是可以配置的。

MySQL 8.0 introduces a delay in the authentication process based on consecutive unsuccessful login attempts. The purpose is to slow down brute force attacks on user passwords. It is possible to configure the number of consecutive unsuccessful attempts before the delay is introduced and the maximum amount of delay introduced.

###### **撤销skip-grant-tables（指远程连接情况下） Retire skip-grant-tables**

8.0版本禁止当实例以`–skip-grant-tables`参数启动时的远程用户连接

MySQL 8.0  disallows remote connections when the server is started with `–skip-grant-tables`.  See also [Bug#79027](https://bugs.mysql.com/bug.php?id=79027) reported by Omar Bourja.

###### **为实例增加mysqld_safe的部分功能 Add mysqld_safe-functionality to server**

8.0版本在实例中引入了部分之前mysqld_safe中的逻辑。可以改善当使用 `--daemonize` 启动参数时在某些情况下的可用性。这也减轻了用户对我们即将移除的`mysqld-safe`脚本的依赖。

MySQL 8.0 implement parts of the logic currently found in the `mysqld_safe` script inside the server. The work improves server usability in some scenarios for example when using the `--daemonize` startup option. The work also make users less dependent upon the `mysqld_safe script`, which we hope to remove in the future. It also fixes [Bug#75343](https://bugs.mysql.com/bug.php?id=75343) reported by Peter Laursen.

### **性能 Performance**

8.0 版本带来更好的读写负载，IO依赖性工作负载，和业务热数据集中的负载。另外新增的资源组特性给用户带来在特定硬件特定负载下将用户线程分配给指定CPU的选项。

MySQL 8.0 comes with better performance for Read/Write workloads, IO bound workloads, and high contention “hot spot” workloads. In addition, the new Resource Group feature gives users an option to optimize for specific workloads on specific hardware by mapping user threads to CPUs.

###### **可伸缩的读写负载 Scaling Read/Write Workloads**

8.0版本对于读写皆有和高写负载的拿捏恰到好处。在集中的读写均有的负载情况下，我们观测到在4个用户并发的情况下，对于高负载，和5.7版本相比有着两倍性能的提高。在5.7上我们显著了提高了只读情况下的性能，8.0则显著提高了读写负载的可扩展性。为MySQL提升了硬件性能的利用率，其改进是基于重新设计了InnoDB写入Redo日志的方法。对比之前用户线程之前互相争抢着写入其数据变更，在新的Redo日志解决方案中，现在Re'do日志由于其写入和刷缓存的操作都有专用的线程来处理。用户线程之间不在持有Redo写入相关的锁，整个Redo处理过程都是时间驱动。。

MySQL 8.0 scales well on RW and heavy write workloads. On intensive RW workloads we observe better performance already from 4 concurrent users  and more than 2 times better performance on high loads comparing to MySQL 5.7. We can say that while 5.7 significantly improved scalability for Read Only workloads, 8.0 significantly improves scalability for Read/Write workloads.  The effect is that MySQL improves  hardware utilization (efficiency) for standard server side hardware (like systems with 2 CPU sockets). This improvement is due to re-designing how InnoDB writes to the REDO log. In contrast to the historical implementation where user threads were constantly fighting to log their data changes, in the new REDO log solution user threads are now lock-free, REDO writing and flushing is managed by dedicated background threads, and the whole REDO processing becomes event-driven.  See blog post by Dimitri Kravtchuk [here](http://dimitrik.free.fr/blog/archives/2017/10/mysql-performance-80-redesigned-redo-log-readwrite-workloads-scalability.html).

###### 榨干IO能力（在更快速的存储设备上） Utilizing IO Capacity (Fast Storage)**

8.0版本允许马力全开的使用存储设备，比如使用英特尔奥腾闪存盘的时候，我们可以在IO敏感的负载情况下获得1百万的采样 QPS（这里说的IO敏感是指不在IBP中，且必须从二级存储设备中获取）。这个改观是由于我们摆脱了 `file_system_mutex`全局锁的争用。

MySQL 8.0 allows users to use every storage device to its full power. For example, testing with Intel Optane flash devices we were able to out-pass 1M Point-Select QPS in a fully IO-bound workload. (IO bound means that data are not cached in buffer pool but must be retrieved from secondary storage). This improvement is due to getting rid of the  `fil_system_mutex` global lock.

###### **在高争用（热点数据）负载情况下的更优性能 Better Performance upon High Contention Loads (“hot rows”)**

8.0版本显著地提升了高争用负载下的性能。高争用负载通常发生在许多事务争用同一行数据的锁，导致了事务等待队列的产生。在实际情景中，负载并不是平稳的，负载可能在特定的时间内爆发（80/20法则）。8.0版本针对短时间的爆发负载无论在每秒处理的事务数（换句话，延迟）还是95%延迟上都处理的更好。对于终端用户来说体现在更好的硬件资源利用率（效率）上。因为系统需要尽量使用榨尽硬件性能，才可以提供更高的平均负载。

MySQL 8.0 significantly improves the performance for high contention workloads. A high contention workload occurs when multiple transactions are waiting for a lock on the same row in a table,  causing queues of waiting transactions. Many real world workloads are not smooth over for example a day but might have bursts at certain hours ([Pareto distributed](https://en.wikipedia.org/wiki/Pareto_distribution)). MySQL 8.0 deals much better with such bursts both in terms of transactions per second, mean latency, and 95th percentile latency. The benefit to the end user is better hardware utilization (efficiency) because the system needs less spare capacity and can thus run with a higher average load. The original patch was contributed by Jiamin Huang ([Bug#84266](https://bugs.mysql.com/bug.php?id=84266)). Please study the [Contention-Aware Transaction Scheduling](https://arxiv.org/pdf/1602.01871.pdf) (CATS) algorithm and read the MySQL blog post by Jiamin Huang and Sunny Bains [here](https://mysqlserverteam.com/contention-aware-transaction-scheduling-arriving-in-innodb-to-boost-performance/).

###### **资源组 Resource Groups**

8.0版本为MySQL引入了全局资源组。有了资源组的概念后，管理人员可以管理用户线程和系统线程对CPU的分配。这个功能可以被用来按CPU分割负载以在某些使用情景下获取更高的效率和性能。因此DBA的工具箱中又多了一把可以帮助自己提升硬件使用率或者增加查询性能的工具。比如在英特尔志强E7-4860 2.27GHz,40核超线程处理器上运行Sysbench读写负载时，我们可以通过将写负载限制在10个核心上从而将总体请求输入量提升一倍。资源组是相当先进的工具，但由于效果会因为负载的类型和手头的硬件的千差万别，这就要求经验经验丰富的管理人员因地制宜地进行使用。

MySQL 8.0 introduces global Resource Groups to MySQL. With Resource Groups, DevOps/DBAs can manage the mapping between user/system threads and CPUs. This can be used to split workloads across CPUs to obtain better efficiency and/or performance in some use cases. Thus, Resource Groups adds a tool to the DBA toolbox,  a tool which can help the DBA to increase hardware utilization or to increase query stability. As an example, with a Sysbench RW workload running on a Intel(R) Xeon (R) CPU E7-4860 2.27 GHz 40 cores-HT box we doubled the overall throughput  by limiting the Write load to 10 cores. Resource Groups is a fairly advanced tool which requires skilled DevOps/DBA to be used effectively as effects will vary with type of load and with the hardware at hand.

## **其他特性 Other Features**

###### **更优的默认值 Better Defaults**

在MySQL了团队中，为了更可能地让用户体验开箱即用，我们对MySQL的默认值保持着紧密的关注。我们已经更改了8.0版本中30多处参数为我们认为更合适的默认参数值。

In the MySQL team we pay close attention to the default configuration of MySQL, and aim for users to have the best out of the box experience possible. MySQL 8.0 has changed more than 30 default values to what we think are better values. See blog post [New Defaults in MySQL 8.0](https://mysqlserverteam.com/new-defaults-in-mysql-8-0/). The motivation for this is outlined in a blog post by Mogan Tocker [here](https://mysqlserverteam.com/planning-the-defaults-for-mysql-5-8/).

###### **协议 Protocol**

8.0版本增加了关闭元数据产生并转化成结果集的选项。构建，解析，发送，接收元数据结果集都会消耗实例，客户端和网络的资源。在某些场景下，元数据大小可能比实际的结果集更大，且不必要。因此在我们关闭了元数据产生和存储后，显著了地提升了查询结果集的转化。客户端如果不想接收于数据一同传回的元数据信息，可以设置 `CLIENT_OPTIONAL_RESULTSET_METADATA` 

MySQL 8.0 adds an option to turn off metadata generation and transfer for resultsets. Constructing/parsing and sending/receiving resultset metadata consumes server, client and network resources. In some cases the metadata size can be much bigger than actual result data size and the metadata is just not needed. We can significantly speed up the query result transfer by completely disabling the generation and storage of these data. Clients can set the `CLIENT_OPTIONAL_RESULTSET_METADATA` flag if they do not want meta-data back with the resultset.

######  **C语言客户端 API C Client API**

8.0版本扩展了 libmysql的C语言API，使其更加稳定地从服务器流式获取复制事务。其设计目的在于为了实现基于binlog的程序（类似Hadoop接收MySQL数据的工具） 进而需要避免对非正式API的调用和对内部头文件的打包

MySQL 8.0 extends libmysql’s C API with a stable interface for getting replication events from the server as a stream of packets. The purpose is to avoid having to call undocumented APIs and package internal header files in order to implement binlog based programs like the MySQL Applier for Hadoop.

###### **内存缓存 Memcached**

8.0版本利用多样的的get操作和对范围查询的支持，提升了InnoDB的内存缓存技能。我们通过对多种get操作的支持还提升了读性能，如用户可以在单次内存缓存查询中获取多个键值对。对范围查询的支持是应脸书的Yoshinori 所请求（译者注：MHA作者）。利用范围查询，用户可以指定一个特定的范围，并获取此区间所有合乎需要的键值。这两个特性对于减少客户端和服务端的来回交互来说都是至关重要的。

MySQL 8.0 enhances the InnoDB Memcached functionalities with *multiple get*operations and support for *range queries*. We added support for the *multiple get*operation to further improve the read  performance, i.e. the user can fetch multiple key value pairs in a single memcached query. Support for *range queries* has been requested by Yoshinori @ Facebook. With range queries, the user can specify a particular range, and fetch all the qualified values in this range. Both features can significantly reduce the number of roundtrips between the client and the server.

###### **持久化的自增计数器 Persistent Autoinc Counters**

8.0版本通过写入redo日志，持久化了自增计数器。持久化的自增计数器解决了一个年代悠久的BUG（译者注：实例在重启后，自增计数器重复利用了之前被使用后删除的自增值）。MySQL恢复进程会重放redo日志，以确保自增计数器的准确数值。不再会出现自增值计数器数值的回滚。这意味着数据库实例在故障恢复时会将redo日志中最后一次发放的自增值作为重建计数器的起点。确保了自增计数器的值不会重复发放，计数器的数值是单调递增的。但要注意可能会有空洞（即发放但是未使用的自增值）。未持久化的自增值在过去一直被认为是一个的有麻烦的故障点。

MySQL 8.0 persists the `AUTOINC` counters by writing them to the redo log. This is a fix for the very old [Bug#199](http://bugs.mysql.com/bug.php?id=199). The MySQL recovery process will replay the redo log and ensure correct values of the `AUTOINC` counters. There won’t be any rollback of `AUTOINC`counters.  This means that database recovery will reestablish the last known counter value after a crash. It comes with the guarantee that the `AUTOINC` counter cannot get the same value twice. The counter is monotonically increasing, but note that there can be gaps (unused values). The lack of persistent `AUTOINC` has been seen as troublesome in the past, e.g. see [Bug#21641](http://bugs.mysql.com/bug.php?id=21641) reported by Stephen Dewey in 2006 or [this](http://mysqlinsights.blogspot.co.uk/2009/03/auto-increment-stability.html) blog post .

## 总结 Summary

综上所述，8.0版本带来了一大堆新特性和性能提升，马上从dev.mysql.com上下载并尝试一下吧(￣▽￣)"。

As shown above, MySQL 8.0 comes with a large set of new features and performance improvements. Download it from [dev.mysql.com](http://dev.mysql.com/downloads/mysql/) and try it out !

当然你也可以从既存的5.7实例升级到8.0版本。在此过程中，你可能需要试试我们随着shell工具包一同下发的新的升级检查工具。这个工具可以帮你检查既存5.7版本的8.0版本升级兼容性（译者附：8.0.3到8.0.11不可以直接升级，或许我用到了旧的工具，改天再试）。可以试试Frédéric Descamps写的t[ Migrating to MySQL 8.0 without breaking old application](http://lefred.be/content/migrating-to-mysql-8-0-without-breaking-old-application/) 博文。

You can also [upgrade](https://dev.mysql.com/doc/refman/8.0/en/upgrading.html) an existing MySQL 5.7 to MySQL 8.0. In the process you might want to try our new [Upgrade Checker](https://mysqlserverteam.com/mysql-shell-8-0-4-introducing-upgrade-checker-utility/) that comes with the new MySQL Shell (mysqlsh). This utility will analyze your existing 5.7 server and tell you about potential 8.0 incompatibilities. Another good resource is the blog post[ Migrating to MySQL 8.0 without breaking old application](http://lefred.be/content/migrating-to-mysql-8-0-without-breaking-old-application/) by Frédéric Descamps.

在这篇文章中我们主要覆盖了服务端的新特性。不仅如此，我们还写了很多关于其他方面的文章，如复制，组复制，InnoDB集群，MySQL Shell,开发API及其相关的连接组件（([Connector/Node.js](https://insidemysql.com/introducing-connector-node-js-for-mysql-8-0/), [Connector/Python](https://insidemysql.com/using-mysql-connector-python-8-0-with-mysql-8-0/), [PHP](https://insidemysql.com/introducing-the-mysql-x-devapi-php-extension-for-mysql-8-0/), [Connector/NET](https://insidemysql.com/introducing-connector-net-with-full-support-for-mysql-8-0/), [Connector/ODBC](https://insidemysql.com/what-is-new-in-connector-odbc-8-0/),[Connector/C++](https://insidemysql.com/what-is-new-in-connector-c-8-0/), and [Connector/J](https://insidemysql.com/connector-j-8-0-11-the-face-for-your-brand-new-document-oriented-database/)）

In this blog post we have covered Server features. There is much more! We will also publish blog posts for other features such as [Replication](https://mysqlhighavailability.com/mysql-8-0-new-features-in-replication/), Group Replication, InnoDB Cluster, [Document Store](https://mysqlserverteam.com/mysql-8-0-announcing-ga-of-the-mysql-document-store/), MySQL Shell, [DevAPI](https://insidemysql.com/mysql-8-0-welcome-to-the-devapi/), and DevAPI based Connectors ([Connector/Node.js](https://insidemysql.com/introducing-connector-node-js-for-mysql-8-0/), [Connector/Python](https://insidemysql.com/using-mysql-connector-python-8-0-with-mysql-8-0/), [PHP](https://insidemysql.com/introducing-the-mysql-x-devapi-php-extension-for-mysql-8-0/), [Connector/NET](https://insidemysql.com/introducing-connector-net-with-full-support-for-mysql-8-0/), [Connector/ODBC](https://insidemysql.com/what-is-new-in-connector-odbc-8-0/),[Connector/C++](https://insidemysql.com/what-is-new-in-connector-c-8-0/), and [Connector/J](https://insidemysql.com/connector-j-8-0-11-the-face-for-your-brand-new-document-oriented-database/)).

就这么多，感谢您使用MySQL

That’s it for now, and **thank you** for using **MySQL** !



注：拖拖拉拉一周终于翻译完，困得要命，手头有其他的事情，暂时没时间精力进行复核，请各位见谅。文章分享在github和blog上，大家可以提request，留言，或者通过QQ微信等工具共同交流，共同学习。