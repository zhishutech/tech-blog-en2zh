原文：http://mysqlserverteam.com/the-mysql-8-0-3-release-candidate-is-available/      
作者：      
译者：琅琊阁    

The MySQL 8.0.3 Release Candidate is available
MySQL8.03 RC 已发布

The MySQL Development team is very happy to announce that MySQL 8.0.3, the first 8.0 Release Candidate (RC1), is now available for download at dev.mysql.com (8.0.3 adds features to 8.0.2, 8.0.1 and 8.0.0). The source code is available at GitHub. You can find the full list of changes and bug fixes in the 8.0.3 Release Notes. Here are the highlights. Enjoy!

MySQL开发团队非常高兴地宣布，第一个8.0 RC版本8.0.3现已可在dev.mysql.com下载（相对于8.0.2，8.0.1和8.0.0，8.0.3添加了一些新特性）。源代码可在GitHub获得。您可以在8.0.3发行说明中看到新版本的改变和bug修复的完整列表。下面是新版本的一些亮点。大家赶快体验吧！

### Histograms
Using histogram statistics in the optimizer (WL#9223) – This work by Erik Froseth makes use of histogram statistics in the optimizer.The primary use case for histogram statistics is for calculating the selectivity (filter effect) of predicates of the form “COLUMN operator CONSTANT”. Optimizer statistics are available to the user through the INFORMATION_SCHEMA.COLUMN_STATISTICS table.

### 直方图
在优化器中使用直方图统计信息(WL#9223)。直方图统计主要使用场景是计算“列运算符常量”形式的谓词的选择性(过滤效果)。用户可以通过INFORMATION_SCHEMA.COLUMN_STATISTICS表获得优化统计信息。

### Force Index
FORCE INDEX to avoid index dives when possible (WL#6526) – This work by Sreeharsha Ramanavarapu allows the optimizer to skip index dives in queries containing FORCE INDEX. An index dive estimates the number of rows. These estimates are used to decide the choice of index. When FORCE INDEX has been specified, this estimation is irrelevant and can be skipped. This applies to a single-table query during execution if FORCE INDEX applies to a single index, without sub-queries, without fulltext index, without GROUP-BY or DISTINCT clauses, and without ORDER-BY clauses.
This optimization applies to range queries and ref. queries, for example a range query like: SELECT a1 FROM t1 FORCE INDEX(idx) WHERE a1 > 'b';. In these cases, an EXPLAIN FOR CONNECTION FORMAT=JSON will output "skip_records_in_range_due_to_force": true and an optimizer trace will output "skipped_due_to_force_index".

### 强制索引
强制索引是为了避免可能发生的index dives(WL#6526)，这个功能可以让优化器在包含FORCE INDEX的查询中跳过index dives，优化器根据index dive预估出来的结果集行数来决定选择走哪个索引。当指定FORCE INDEX时，这个预估就变得无关紧要，可以跳过。这个功能适用于单表查询时FORCE INDEX作用于单个索引，不含子查询、全文索引，GROUP BY、DISTINCT，ORDER BY等多种情况。
此优化应用于范围查询和普通索引查询。例如范围查询，如：SELECT a1 FROM t1 FORCE INDEX（idx）WHERE a1>'b'。这种情况下，EXPLAIN FOR CONNECTION FORMAT = JSON将输出skip_records_in_range_due_to_force”：true，并且optimizer trace将输出“skipped_due_to_force_index”。

### Hints
Hint to temporarily set session variable for current statement (WL#681) – This work by Sergey Glukhov implements a new optimizer hint called SET_VAR. The SET_VAR hint will set the value for a given system variable for the next statement only. Thus the value will be reset to the previous value after the statement is over. SET_VAR covers a subset of sessions variables, since some session variables either do not apply to statements or must be set at an earlier stage of statement execution. Some variable settings have much more meaning being set for a query rather than for a connection, for example you might want to increase sort_buffer_size before doing a large sort query but it is very likely other queries in a session are simple and you are quite OK with default settings for those. E.g.SELECT /*+ SET_VAR(sort_buffer = 16M) */ name FROM people ORDER BY name;

### Hints
优化器支持了一个新的hint，在当前语句中使用SET_VAR语法临时设置会话变量(WL#681)，SET_VAR只为下一个语句的给定系统变量设置值。因此，在语句结束后，该变量将被重置为先前的值。 SET_VAR涵盖会话变量的一个子集，因此一些会话变量不适用于该语句，或者必须在语句执行的早期阶段设置。一些变量的调整可能对于查询的意义比连接本身更大，例如您可能希望在进行大型排序查询之前增加sort_buffer_size，但很可能会话中的其他查询却很简单，这就可以使用默认设置。用法例如：SELECT / * + SET_VAR（sort_buffer = 16M）* / name FROM people ORDER BY name;

### Invisible Indexes
Optimizer switch to see invisible indexes(WL#10891) – This work by Martin Hansson implements an optimizer switch named use invisible indexes which can be turned ON or OFF (default). Optimizer switches can be activated on a per session basis by SET @@optimizer_switch='use_invisible_indexes=on'; This feature supports the use case where a user wants to roll out an index. For example, the user may want to create the index as invisible and then activate the index in a specific session to measure the effect.

### 不可见索引
不可见索引可以被设置为ON或者OFF(默认)，优化器可以根据其状态切换模式选择是否走不可见索引。优化器的模式可以通过SET @@optimizer_switch='use_invisible_indexes=on';语句实现基于会话级别的切换。此功能支持用户想要测试索引的用例。例如，用户可能希望将索引创建为不可见，然后在特定会话中激活索引来测试效果。

### Common Table Expressions
Limit recursion in CTE (WL#10972) – This work by Guilhem Bichot implements a global and session variable called cte_max_recursion_depth to limit recursion in CTEs (default 1000, min 0, max 4G).This is done to protect the users from runaway queries, for example if the user forgets to add a WHERE clause to the recursive query block. When a recursive CTE does more than cte_max_recursion_depth iterations, the execution will stop and return an error message. iterations, the execution will stop and return an error message.

### 通用表表达式
限制CTE中的递归。用一个名为cte_max_recursion_depth的变量（默认为1000，最小0，最大4G）来限制CTE中的递归。这样做是为了保护用户失控查询，例如，如果用户忘记向递归查询块添加WHERE子句。当递归CTE执行超过cte_max_recursion_depth次时，执行将停止并返回错误消息。

### Character Sets
Add Russian collations for utf8mb4 (WL#10753) – This work by Xing Zhang adds Russian collations utf8mb4_ru_0900_ai_ci and utf8mb4_ru_0900_as_cs for character set utf8mb4. The new collations sort characters of Russian language according to language specific rules defined by Unicode CLDR.

### 字符集
针对utf8mb4增加了俄罗斯的校对规则utf8mb4_ru_0900_ai_ci和utf8mb4_ru_0900_as_cs，新的归类根据Unicode CLDR定义的语言特定规则对俄语的字符进行排序。

### JSON
- Add JSON_MERGE_PATCH, rename JSON_MERGE to JSON_MERGE_PRESERVE (WL#9692) –This work by Knut Anders Hatlen implements two alternative JSON merge functions,JSON_MERGE_PATCH() and JSON_MERGE_PRESERVE().
- The JSON_MERGE_PATCH() function implements the semantics of JavaScript (and other scripting languages) specified by RFC7396, i.e. it removes duplicates by precedence of the second document. For example, JSON_MERGE('{"a":1,"b":2 }','{"a":3,"c":4 }');#returns{"a":3,"b":2,"c":4}.
- JSON_MERGE_PRESERVE（）函数和MySQL 5.7中的JSON_MERGE（）是一样的，它会保留所有的值，例如JSON_MERGE('{"a":1,"b":2}','{"a":3,"c":4}');# returns {"a":[1,3],"b":2,"c":4}.
- The existing JSON_MERGE() function is deprecated in MySQL 8.0 to remove ambiguity for the merge operation. See also proposal in Bug#81283.

### JSON
- 提供了两个可选择的JSON的合并函数，JSON_MERGE_PATCH（新添加）和JSON_MERGE_PRESERVE（原JSON_MERGE的重命名）。
- JSON_MERGE_PATCH（）函数通过RFC7396实现了JavaScript（和其他脚本语言）的语法，换句话说，它将按照第二个文档为高优先级删除重复项。例如：JSON_MERGE('{"a":1,"b":2 }','{"a":3,"c":4 }'); # returns{"a":3,"b":2,"c":4}
- The JSON_MERGE_PRESERVE() function has the semantics of JSON_MERGE() implemented in MySQL 5.7 which preserves all values, for example JSON_MERGE('{"a":1,"b":2}','{"a":3,"c":4}'); #returns{"a":[1,3],"b":2,"c":4}.
- 为了消除合并操作的歧义，之前的JSON_MERGE（）函数在MySQL 8.0中已被弃用。参见Bug＃81283中的提案。

### GIS
- Support SRID in InnoDB Spatial Index(WL#10439) – This work by Elzbieta Babij makes InnoDB Spatial Indexes aware of the Spacial Reference System (SRS) of the indexed column.The geography support in 8.0 needs to compare geometries using different formulas depending upon the SRS. Therefore, the index must know which SRS it is in in order to work correctly. When a spatial index is created InnoDB will do a sanity check that SRID for all rows are of the same type as specified by column.See Argument Handling by Spatial Functions.
- Ellipsoidal R-tree support functions (WL#10827) – This work by Norvald Ryeng reimplements the R-trees support functions for Minimum Bounding Rectangles (MBR) operations in a way that supports both Cartesian and geographical computations. R-tree indexes on columns with Cartesian geometries use Cartesian computations, and R-tree indexes on columns with geographic geometries use geographic computations. If an R-tree contains a mix of Cartesian and geographic geometries, or if any geometries are invalid, the result of any operation on that index is undefined.
- SRID type modifier for geometric types (WL#8592) – This work by Erik Froseth adds a new column property for geometric types to specify the SRID. For example SRID 4326 in CREATE TABLE t1 (g GEOMETRY SRID 4326, p POINT SRID 0 NOT NULL);. Values inserted into a column with an SRID property must be in that SRID. Attempts to insert values with other SRIDs results in an exception condition being raised. Unmodified types, i.e., types with no SRID specification, will continue to accept all SRIDs as before. The optimizer is changed so that only indexes on columns with the SRID specified will be considered in query planning/execution.The specified SRID is exposed in both INFORMATION_SCHEMA.GEOMETRY_COLUMNS and INFORMATION_SCHEMA.COLUMNS.

### GIS
- InnoDB中地理空间索引（WL＃10439）支持SRID ,使得InnoDB地理空间索引知道索引列的空间参考系（SRS）。8.0中的地理空间支持需要根据SRS使用不同公式来对比几何位置。因此，索引必须知道其在哪个SRS中才能正常工作。当创建地理空间索引时，InnoDB将进行一个完整性检查，确保所有行的SRID与列指定的类型相同。参见空间函数的参数处理。
- 支持椭圆形R树功能（WL＃10827），R树支持用笛卡尔和地理计算的方式重新实现了最小边界矩形（MBR）操作的功能。R-tree索引上的笛卡尔几何和地理几何分别用笛卡尔和地理计算，如果R-tree包含笛卡尔和地理几何的混合，或者如果任意一个几何形状无效，则该索引上的任何操作的结果都是未定义的。 
- 几何类型的SRID类型修改（WL＃8592） 为几何类型添加了一个新的列属性来指定SRID。 例如CREATE TABLE t1（g GEOMETRY SRID 4326，p POINT SRID 0 NOT NULL）中的SRID 4326;插入具有SRID属性的列中的值必须在该SRID中。尝试使用其他SRID插入值会导致引发异常情况。未修改的类型也就是不具有SRID规范的类型，还像以前一样继续接受所有的SRID。这个优化的改变方便了在查询计划/执行中仅考虑指定SRID的索引列的情况。指定的SRID显示在INFORMATION_SCHEMA.GEOMETRY_COLUMNSINFORMATION_SCHEMA.COLUMNS中。

### Resource Groups
Resource Groups (WL#9467) – This work by Thayumanavar Sachithanantha introduces global Resource Groups to MySQL. The purpose of Resource Groups is to decide on the mapping between user/system threads and CPUs. This can be used to split workloads across CPUs to obtain better efficiency and/or performance in some use cases. There are two default groups, one for user threads and one for system threads. Both default groups have 0 priority and no CPU affinity. DevOps/DBAs can create and manage additional Resource Groups with priority and CPU affinity using SQL CREATE/ALTER/DROP RESOURCE GROUP. Information about existing resource groups are found in INFORMATION_SCHEMA.RESOURCE_GROUPS. The user can execute a SQL query on a given resource group by adding the hint /*+ RESOURCE_GROUP(resource_group_name) */  after the initial SELECT, UPDATE, INSERT, REPLACE or DELETE keyword.

### 资源组
资源组(WL#9467)决定了用户/系统线程和CPU之间的映射用于在CPU之间拆分工作负载，以便在某些用例中获得更好的效率、性能。默认有两个资源组，一个用于用户线程，一个用于系统线程。两个默认的资源组优先级都为0，没有CPU亲缘性。DevOps / DBA可以使用SQL CREATE / ALTER / DROP RESOURCE GROUP语句创建和管理具有优先级和CPU亲和力的额外的资源组。有关现有资源组的信息，请参见INFORMATION_SCHEMA.RESOURCE_GROUPS。用户可以通过在SELECT，UPDATE，INSERT，REPLACE或DELETE关键字之后添加hint “/ * + RESOURCE_GROUP（resource_group_name）* / ” 来对给定的资源组执行SQL查询 

### Performance Schema
- Digest Query Sample (WL#9830) – This work by Christopher Powers makes some changes to the EVENTS_STATEMENTS_SUMMARY_BY_DIGEST performance schema table to capture a full example query and some key information about this query example. The column QUERY_SAMPLE_TEXT is added to capture a query sample so that users can run EXPLAIN on a real query and to get a query plan. The column QUERY_SAMPLE_SEEN is added  to capture the query sample timestamp. The column QUERY_SAMPLE_TIMER_WAIT is added to capture the query sample execution time. The columns FIRST_SEEN and LAST_SEEN  have been modified to use fractional seconds.
- Instrumentation meta-data (WL#7801) – This work by Marc Alff adds meta-data for instruments in performance schema table SETUP_INSTRUMENT. Meta-data act as online documentation, to be looked at by users or tools. It adds columns for “properties”, “volatility”, and “documentation”.
- Service for componets (WL#9764) – This work by Marc Alff exposes the existing performance schema interface as a service which can be consumed by components in the new service infrastructure. With this work, code compiled as a component (not as a “plugin”) can invoke the performance schema instrumentation.

### Performance Schema
- 对EVENTS_STATEMENTS_SUMMARY_BY_DIGEST性能模式表进行了一些更改，以捕获完整示例查询和有关此查询示例的一些关键信息。添加QUERY_SAMPLE_TEXT列以捕获查询示例，以便用户可以在真实查询上运行EXPLAIN并获取查询计划。添QUERY_SAMPLE_SEEN列以捕获查询样本时间戳。添加QUERY_SAMPLE_TIMER_WAIT列以捕获查询示例执行时间。FIRST_SEEN列和LAST_SEEN 列已被修改为小数秒表示。
- 为P_S中SETUP_INSTRUMENT中的instruments添加元数据。元数据作为在线文档，由用户或工具查看。添加了三个列分别是“properties”, “volatility”, and和“documentation”。
- 将现有的性能架构接口公开为新服务基础架构中组件可以使用的服务。代码被编译为组件（而不是“插件”）以支持调用P_S的instrumentation。**(这里的instrumentation不知道怎么理解)**

### Security
- Caching sha2 authentication plugin (WL#9591) – This work by Harin Vadodaria introduces a new authentication plugin, caching_sha2_password, which uses a caching mechanism to speed up authentication.
- A password is created (CREATE/ALTER USER) over a TLS protected connection.  The server does multiple rounds of SHA256(password) with SALT and stores the “expensive” HASH in mysql.user.authentication_string. Note that only the expensive HASH is stored on server side, not the password. Then the “new user” initiates a authentication and gets a random number back from the server. The client sends a HASH based on this random number and the user provided password back to the server. The server then tries to verify the received HASH against the cached entry. If it is in the cache the authentication is ok, but the first time the user connects it will not be in the cache.
- When the entry is not in the cache a full “expensive authentication” takes place: The client sends password on a TLS connection or encrypts the password using RSA keypair (password never sent without encryption). The server decrypts the password and creates the expensive HASH and compares it with mysql.user.authentication_string. If there is no match the server returns an error to the client, otherwise it creates a fast HASH and stores it in cache entry : ‘user’@’host’ -> SHA256(SHA256(password)) and returns ok back to the client. The next time the user is authenticated it will be fast since it will find the entry in the cache.
- Password rotation policy (WL#6595) – This work by Georgi Kodinov introduces restrictions on password reuse. Restrictions can be configured at global level as well as individual user level. Password history is kept secure because it may give clues about habits or patterns used by individual users when they change their password. As previously, MySQL offers a password expiration policy which enforces password change based on time. MySQL also has the ability to control what can and can not be used as password. This work restrict password reuse and thus forces users to supply new strong passwords with each password change.
- Retire skip-grant-tables (WL#4321) – This work by  Kristofer Älvring disallows remote connections when the server is started with –skip-grant-tables.  See also Bug#79027 reported by Omar Bourja.

### 安全
- 引入了一个新的认证插件caching_sha2_password，它使用缓存机制来加快身份验证。
- 通过TLS保护的连接创建密码（CREATE / ALTER USER）。服务使用SALT执行多轮SHA256（密码），并将“expensive”HASH存储在mysql.user.authentication_string中。请注意，只有昂贵的HASH存储在服务端，而不是密码。然后“新用户”启动身份验证，并从服务端获取随机数。客户端根据该随机数发送一个HASH，并将用户提供的密码发送回服务端。然后，服务端尝试根据缓存的条目验证接收到的HASH。如果是在缓存中，认证就可以，但用户第一次连接它不会在缓存中
- 当验证信息不在缓存中时，将发生完整的“昂贵的身份验证”：客户端在TLS连接上发送密码，或使用RSA密钥对密码密码（密码从未发送，无加密）。服务端解密密码并创建昂贵的HASH并与mysql.user.authentication_string进行比较。如果没有匹配，服务端会向客户端返回错误，否则创建一个快速的HASH并将其存储在缓存条目：'user'@'host' - > SHA256（SHA256（password））中，并返回到客户端。下次用户认证时，它会很快，因为它会在缓存中找到该验证信息。
- 引入了对密码重用的限制。可以在全局级别以及单个用户级别配置限制。因为它可能会提供个人用户更改密码时使用的习惯或模式的线索，所以密码将保持安全。就像之前，MySQL提供密码到期策略，密码到期时强制更改密码。MySQL也有能力控制什么内容可以和不能用作密码。限制了密码重用，从而迫使用户在每个密码更改时提供新的强密码。
- 当使用-skip-grant-tables启动服务时，将不允许远程连接，另见Omar Bourja的 Bug#79027报告

### Protocol
Make metadata information transfer optional (WL#8134) – This work by Ramil Kalimullin adds an option to turn off metadata generation and transfer for resultsets. Constructing/parsing and sending/receiving resultset metadata consumes server, client and network resources. In some cases the metadata size can be much bigger than actual result data size and the metadata is just not needed. We can significantly speed up the query result transfer by completely disabling the generation and storage of these data. This work introduces a new session variable called resultset_metadata which can either be FULL (default) or NONE. Clients can set the CLIENT_OPTIONAL_RESULTSET_METADATA flag if they do not want meta data back with the resultset. There are no protocol changes for clients that don’t set the CLIENT_OPTIONAL_RESULTSET_METADATA, such clients will operate as before.

### 协议
增加了关闭元数据生成和转移结果集的选项。构造/解析和发送/接收结果集元数据消耗服务端，客户端和网络资源。在某些情况下，元数据大小可能远远大于实际的结果数据大小，并且这个元数据是没必要的。通过完全禁用这些数据的生成和存储，可以明显加快查询结果传输速度。引入了一个新的会话级的变量叫resultset_metadata，可以设置 FULL（默认）或NONE。如果不想使用结果集返回元数据，用户可以设置CLIENT_OPTIONAL_RESULTSET_METADATA标记。如果没有设置CLIENT_OPTIONAL_RESULTSET_METADATA，客户端将和先前一样运行，不涉及协议的变更。

### Service Infrastructure
- Component status variables as a service for mysql_server component (WL#10806) – This work by Venkata Sidagam provides a status variable service to components by the mysql_server component. The components can register, unregister, and get_variable to handle their own status variables. The component status variables will be added as status variables to the global names space of status variables.
- Configuration system variables as a service for mysql_server component (WL#9424) – This work by Venkata Sidagam provides a system variable service to components by the mysql_server component. The components can register, unregister, and get_variable to handle their own system variables. The component system variables will be added as status variables to the global names space of system variables.

### 服务基础设施
- 由mysql_server组件提供组件的状态变量服务。组件可以注册，注销和get_variable来处理自己的状态变量。组件状态变量将作为状态变量添加到状态变量的全局名称空间。
- 通过mysql_server组件为组件提供系统变量服务。组件可以注册，注销和get_variable来处理自己的系统变量。组件系统变量将作为状态变量添加到系统变量的全局名称空间。

### X Protocol / X Plugin
- mysqlx.Crud.Update with MERGE_PATCH (WL#10797) – This work by Lukasz Kotula adds an operation type called MERGE_PATCH to the X Protocol Mysqlx.Crud.Update message. When the X Plugin handles the Mysqlx.Crud.Update message it uses the JSON_MERGE_PATCH() function in the server to modify documents. A document patch expression contains instructions about how the source document is to be modified producing a derived document.  Document patches are represented as Mysqlx.Expr.Expr objects, as already defined in the X protocol.  SQL mapping takes advantage of the JSON_MERGE_PATCH() function, which has the desired semantics for merging a “patch” document against another JSON document. In short, the mapping takes the form of: @result = JSON_MERGE_PATCH(source, @patch_expr) where @patch_expr is the expression generated for the patch object.
- Mysqlx.Crud.Update on top level document  (WL#10682) – This work by Grzegorz Szwarc modifies the existing Mysqlx.Crud.Update operation in the X Protocol / X Plugin. With this change the update operations (ITEM_REMOVE, ITEM_SET, ITEM_REPLACE, ARRAY_INSERT, ARRAY_APPEND) allow an empty document path to be specified. An empty document-path means that the update operates on the whole document.  In other words, all operations that are executed through Mysqlx.Crud.Update can now operate on whole/root document. Any operation done on an existing document will preserve  the existing Document ID.
- Mysqlx.Crud.Find with row locking  (WL#10645) – This work by Tomasz Stepniak adds a “locking” field to the Mysqlx.Crud.Find message. The X Plugin interprets the value under “locking” to activate the innodb locking functionality. There are three cases possible: 1) “locking” field was not specified, the interpretation of the message is the same as in old plugin (no locking activated). 2) “locking” was set to “SHARED_LOCK”, the interpretation of the message adds “LOCK IN SHARE MODE” to the generated SQL (triggering the innodb  locking functionality), and 3) “locking” was set to “EXCLUSIVE_LOCK”, the interpretation of the message adds “FOR UPDATE” to the generated SQL (triggering the innodb locking functionality).
- Spatial index type  (WL#10734) – This work by Grzegorz Szwarc adds support for spatial indexes on GeoJSON data stored in JSON documents. Geographical coordinates in a document collection are represented in the GeoJSON format. GeoJSON data is converted to the GEOMETRY datatype by the X Plugin. GEOMETRY datatypes can be indexed by spatial indexes.
- Full-Text index type  (WL#10744) – This work by Grzegorz Szwarc makes adds support for Full-Text indexes on Documents. Full-Text indexes allows searching the entire document (or a sub-document) for any text value.
- X Protocol expectations for supported protobuf fields  (WL#10237) – This work by Lukasz Kotula introduces a new “condition key” to the X Protocol expectation mechanism. The client is going to send an “Expect Open” message containing message/fields tag chain and the server is going to validate if the field specified this way is present inside the definition of servers X Protocol message. This is done to ensure pipelining, message processing should be stopped when any message does not meet the expectation. This functionality helps to detect compatibility problems between the client application and the MySQL Server, when the server receives an X Protocol message containing a field that it doesn’t know.
- X Protocol connector code extraction from mysqlxtest to libmysqlxclient  (WL#9509) – This work by Lukasz Kotula implements a low level client/connector library that is going to be used by both internal and external components to connect to MySQL Server using the X Protocol. Hence the libmysqlxclient plays a similar role for X Protocol as libmysqlclient has done for the classic protocol.

### X 协议/X 插件
- 向X协议Mysqlx.Crud.Update消息中添加一个名为MERGE_PATCH的操作类型。当X插件处理Mysqlx.Crud.Update消息时，它使用服务中的JSON_MERGE_PATCH（）函数来修改文档。文档补丁表达式包含有关如何修改源文档以生成派生文档的说明。文档补丁表示为Mysqlx.Expr.Expr对象，表示已在X 协议中定义过(这句有点问题)。SQL映射利用JSON_MERGE_PATCH（）函数，它具有将“补丁”文档与另一个JSON文档合并所需的语义。简而言之，映射的形式为：@result = JSON_MERGE_PATCH（source，@patch_expr）其中@patch_expr是为该补丁对象生成的表达式。
- 支持修改X协议/ X插件中现有的Mysqlx.Crud.Update操作。通过此更改，更新操作（ITEM_REMOVE，ITEM_SET，ITEM_REPLACE，ARRAY_INSERT，ARRAY_APPEND）允许指定一个空文档路径。空文档路径意味着更新对整个文档进行操作。换句话说，通过Mysqlx.Crud.Update执行的所有操作现在可以在整个/ root文档上运行。对现有文档进行的任何操作都将保留现有的文档ID。
- 向Mysqlx.Crud.Find消息添加了一个“locking”字段。X插件解析“locking”状态下的值以激活innodb锁定功能。有三种情况可能：1）“locking”字段未指定，消息的解析与旧插件（无锁定激活）相同。2）“locking”设置为“SHARED_LOCK”，消息的解析在生成的SQL中添加“LOCK IN SHARE MODE”（触发innodb锁定功能），3）“locking”设置为“EXCLUSIVE_LOCK”，对消息的解析为生成的SQL添加“FOR UPDATE”（触发innodb锁定功能）。
- 增加了对存储在JSON文档中的GeoJSON数据的空间索引的支持。文档集合中的地理坐标以GeoJSON格式表示。GeoJSON数据由X插件转换为GEOMETRY数据类型。可以通过空间索引对GEOMETRY数据类型进行索引。
- 增加了对Document上全文索引的支持。全文索引允许搜索整个文档（或子文档）以获取任何text值。
- 向X协议期望机制引入了一个新的“条件密钥” 。客户端将发送包含消息/字段标签链的“预期打开”消息，如果以这种方式指定的字段存在于服务的X协议消息的定义内，服务端则将进行验证。这是为了确保管道中当任何消息不符合期望时，停止消息处理。当服务收到包含不知道的字段的X协议消息时，此功能有助于检测客户端应用程序和MySQL服务之间的兼容性问题。
- 实现了一个低级别的客户端/连接器库通过内部和外部组件连接MySQL服务都将使用X协议。因此，libmysqlxclient与libmysqlclient对于经典协议所做的X协议起着类似的作用。

### Performance
Use CATS for scheduling lock release under high load (WL#10793) – This work by Sunny Bains implements Contention-Aware Transaction Scheduling (CATS) in InnoDB. The original patch was contributed by Jiamin Huang (Bug#84266). CATS helps in reducing the lock sys wait mutex contention by granting locks to transactions that have a higher wait in the dependency graph. The implementation keeps track of how many transactions are waiting for locks that are already acquired by a transaction and, recursively, how many transaction are waiting for those waiting transactions in the wait for graph. The waits-for-edge is “weighted” and this weight is used to order the transactions when scheduling the lock release. The weight is a cumulative weight of the dependencies.

### 性能
在InnoDB中实现了竞争意识事务调度（CATS）。原来的补丁是Jiamin Huang（Bug＃84266）提供的。CATS有助于通过向依赖关系图中具有较高等待的事务授予锁来减少锁等待互斥竞争。实现了记录跟踪事务已经获取的锁的数量有多少，进一步说，就是获取在等待图中等待其他事务的事务数量是多少。waits-for-edge是“加权”的，这个权重用于在调度锁版本时对事务进行排序。权重是依赖关系的累积权重。

### Tablespaces
- InnoDB: Stop using rollback segments in the system tablespace (WL#10583) – This work by Kevin Lewis changes the minimum value for innodb_undo_tablespaces to 2 and modifies the code that deals with rollback segments in the system tablespace so that it can read, but not create or update rollback segements in an existing system tablespace. In 8.0, rollback segements are moved out of the system tablespace and into UNDO tablespaces.
- Rename a general tablespace  (WL#8972)  – This work by Dyre Tjeldvoll implements ALTER TABLESPACE s1 RENAME TO s2; A general tablespace is a user-visible entity which users can CREATE, ALTER, and DROP. See also Bug#26949, Bug#32497, and Bug#58006.

### 表空间
- 将innodb_undo_tablespaces的最小值更改为2，并修改处理系统表空间中的回滚段的代码，以便它可以读取在现有系统表空间中创建或更新回滚段。在8.0中，回滚段从系统表空间挪到了UNDO 表空间中了。
- 实现了使用ALTER TABLESPACE s1 RENAME TO s2;重命名通用表空间，通用表空间是用户可见的实体，用户可以使用CREATE，ALTER和DROP操作它。另请参见Bug＃26949，Bug＃32497和Bug＃58006。

### DDL
ALTER TABLE RENAME COLUMN (WL#10761) – This work by Abhishek Ranjan implements ALTER TABLE ... RENAME COLUMN old_name TO new_name;. This is an improvement over existing syntax ALTER TABLE <table_name> CHANGE ... which requires re-specification of all the attributes of the column. The old/existing syntax has the disadvantage that all the column information might not be available to the application trying to do the rename. There is also a risk of accidental data type change in the old/existing syntax which might result in data loss.

### DDL
实现了ALTER TABLE ... RENAME COLUMN old_name TO new_name;更改列名。老的语法ALTER TABLE <table_name> CHANGE ...需要重新定义所有列的属性，这方面来看是一个提升，老的语法的缺点是可能不是所有的列信息都能被应用程序重命名（这句有点问题）。在老的语法中还存在额外的数据类型更改的风险，这可能导致数据丢失。

### Replication
Replication of partial JSON updates (WL#2955) – This work by Maria Couceiro implements an option to enable/disable partial JSON updates. If the user disables the option, the server must only write full JSON documents to the binary log, and never partial JSON updates. If the user enables the option, the server may write partial JSON updates to the after-image of Row Based Replication updates in the binary log when possible. It may also write full JSON documents, e.g. in case the server cannot generate a partial JSON update, or if the partial JSON update would be bigger than the full document.

### 复制
实现了启用/禁用部分JSON更新的选项。如果用户禁用该选项，则服务只能将完整的JSON文档写入二进制日志，而不能部分地进行JSON更新。如果用户启用该选项，则基于行的复制中服务可能会将部分JSON更新写入二进制日志中的的后像。它也可以写完整的JSON文档，例如，如果服务器无法生成部分JSON更新，或者部分JSON更新将大于完整文档的时候。

### Group Replication
- Instrument threads in GCS/XCom (WL#10622) – This work by Filipe Campos instruments the GCS and XCom threads and exposes them automatically in performance schema table metrics. It is also a requirement that we do further instrumentation in XCom and GCS, such as mutexes and condition variables, as well as memory usage.
- Change GCS/XCOM to have dynamic debugging and tracing (WL#10200) – This work by Alfranio Correia implements dynamical filtering for debugging and tracing messages per sub-system (i.e. GCS, XCOM, etc). Debugging can be turned on by SET GLOBAL group_replication_communication_debug_options='GCS_DEBUG_ALL';. Error, warning and information messages will be output as defined by the server’s error logging component. Debug and trace messages will sent to a file when group replication is in use. By default the file used as debug sink will be named GCS_DEBUG_TRACE and will be placed in the data directory.

### 组复制
- 对GCS和XCom线程进行了调整，并在performance schema metrics中自动显示。还要求我们在XCom和GCS中进行进一步的测试，例如互斥体和条件变量以及内存使用。
- 对每个子系统（即GCS，XCOM等）进行调试和跟踪消息的动态过滤。可以通过SET GLOBAL group_replication_communication_debug_options ='GCS_DEBUG_ALL';打开调试  。错误，警告信息将按服务的错误记录组件定义输出。使用组复制时，调试和跟踪消息将发送到文件。默认情况下，用作调试接收信息的文件将被命名为GCS_DEBUG_TRACE，并将被放置在数据目录中。

### Data Dictionary
- Support crash-safe DDL (WL#9536) – This work by Bin Su and Jimmy Yang ensures crash-safe DDL for MySQL. This work materializes one of the main benefits of having one common transactional data dictionary for server and storage engine layers, i.e. it is no longer possible for Server and InnoDB to have different metadata for database objects.
- Improve crash-safety of non-table DDL (WL#9173) – This work by  Praveenkumar Hulakund ensures crash-safety of non-table DDLs. For example CREATE/ALTER/DROP FUNCTION/PROCEDURE/EVENT/VIEW.
- Implicit tablespace name should be same as table name (WL#10436)  – This work by Thirunarayanan Balathandayuth ensures that a CREATE TABLE <tablename> creates an implicit tablespace with the same name. This is only for implicitly created tablespaces, the user can also create explicitly named tablespaces and create tables within explicitly named tablespaces.
- Remove InnoDB system tables and modify the views of their information schema counterparts (WL#9535)  – This work by Zheng Lai removes the InnoDB internal data dictionary (SYS_* tables). Some of the INFORMATION_SCHEMA information are based on SYS_* tables, for example information_schema.innodb_sys_tables and information_schema.innodb_sys_tablespaces. These information_schema tables are replaced with views over data dictionary tables.
- Integrating InnoDB SDI with new data dictionary (WL#9538)  – This work by Satya Bodapati ensures that the JSON formatted Serialized Dictionary Information (SDI) is stored in the InnoDB tablespaces. This work also assures that the SDI gets updated when meta-data are changed, e.g. because of an ALTER TABLE. There is also a tool ibd2sdi, which is able extract SDI from an InnoDB tablespace when the server  is offline.
- Implement INFORMATION_SCHEMA system views for FILES/PARTITIONS (WL#9814)  – This work by Gopal Shankar implements new system views definition for  INFORMATION_SCHEMA.PARTITIONS and INFORMATION_SCHEMA.FILES. These views read metadata directly from data dictionary tables.
- Meta-data locking for FOREIGN KEY tables (WL#6049) – This work by  Dmitry Lenev implements meta-data locking for foreign keys. This involves acquiring metadata locks on tables across foreign key relationships so that conflicting operations are blocked as well as updating FK metadata if a parent table changes. This work is enabled by the common data dictionary which makes foreign keys visible to the server layer, thus to meta-data locking.
- Implement INFORMATION_SCHEMA.REFERENTIAL_CONSTRAINTS as a system views over dictionary tables (WL#11059) – This work by Gopal Shankar implements new system views definition for INFORMATION_SCHEMA.REFERENTIAL_CONSTRAINTS. This view reads metadata directly from data dictionary tables.

### 数据字典
- 支持DDL的crash-safe。实现了为服务器和存储引擎层提供一个常用的事务数据字典的主要优点之一，即Server和InnoDB不再可能为数据库对象拥有不同的元数据。
- 提高了非表级别的DDL操作崩溃的安全性，确保非表级别DDL的安全。例如CREATE / ALTER / DROP FUNCTION / PROCEDURE / EVENT / VIEW。
- 确保隐式表空间和表名一致。这只是对于隐式创建的表空间,用户还可以创建显式指定的表空间和显式指定的表空间内创建表。
- 删除了InnoDB内部数据字典（SYS_ *表）。一些INFORMATION_SCHEMA信息基于SYS_ *表，例如information_schema.innodb_sys_tables和information_schema.innodb_sys_tablespaces。这些information_schema表将替换为数据字典表中的视图。
- 将InnoDB SDI与新的数据字典整合在一起，确保JSON格式的序列化字典信息（SDI）存储在InnoDB表空间中。还保证当元数据更改时，SDI会被更新，例如ALTER TABLE操作。还有一个工具ibd2sdi，当服务关闭时，它可以从InnoDB表空间中提取SDI。
- 实现了元数据锁定。这涉及跨外键关系获取表上的元数据锁，以便阻止如果父表更改则更新FK元数据的冲突操作。这个工作是由通用数据字典启用的，这使得外键对server层可见，从而进行元数据锁定。
- 为INFORMATION_SCHEMA.PARTITIONS和INFORMATION_SCHEMA.FILES实现了新的系统视图定义  。这些视图直接从数据字典表中读取元数据。
- 为INFORMATION_SCHEMA.REFERENTIAL_CONSTRAINTS实现了新的系统视图定义。此视图从数据字典表中直接读取元数据。

### MTR Tests
- Change Server General tests to run with new default charset (WL#10299) – This work by Deepa Dixit fixes MTR tests so they now run with the new default character set.
- Add/Extend mtr tests for Replication/GR for roles (WL#10886) – This work by Deepthi E.S. adds MTR tests to ensure that Roles are replicated as expected. For example tests must verify that ROLES on replication users used in ‘CHANGE MASTER TO’work as expected for RPL/GR.
- Add/Extend mtr tests for replication with generated columns and X plugin (WL#9776) – This work by Parveez Baig adds MTR tests to ensure that stored/virtual columns are replicated as expected. For example that replication shall not be affected when transactions involve stored or virtual columns.

### MTR测试
- 为了修复MTR测试，所以现在这些测试用的是新的默认字符集。
- 添加/扩展用于角色复制/ GR的mtr测试，以确保角色按预期复制。例如，测试必须验证复制用户上的ROLES在“change master to”中是否符合RPL / GR的预期。
- 使用生成的列和X插件添加/扩展复制的mtr的测试，以确保真实/虚拟列按预期方式进行复制。例如，当事务涉及真实列或虚拟列时，不会影响复制。


### Library Upgrade
Upgrade zlib libraries to 1.2.11 in trunk (WL#10551) – This work by Aditya A. upgrades the zlib library versions from zlib 1.2.3 to zlib 1.2.11 for MySQL 8.0.

###库升级
MySQL 8.0中，zlib库从1.2.3版本升级到1.2.11版本。


### Changes to Defaults
Autoscale InnoDB resources based on system resources by default (WL#9193) – This work by Mayank Prasad introduces a new option innodb_dedicated_server which can be set OFF/ON (OFF by default). If ON, settings for following InnoDB variables (if not specified explicitly) would be scaled accordingly innodb_buffer_pool_size, innodb_log_file_size,and innodb_flush_method. See also blog post Plan to improve the out of the Box Experience in MySQL 8.0 by Morgan Tocker.
- Change innodb_autoinc_lock_mode default to 2 (WL#9699) – This work by Mayank Prasad changes the default of innodb_autoinc_lock_mode from sequential (1) to interleaved (2). This can be done because the default replication format is row-based replication. This change is known to be incompatible with statement based replication, and may break some applications or user-generated test suites that depend on sequential auto increment. The previous default can be restored by setting innodb_autoinc_lock_mode=1;
- Change innodb_flush_neighbors default to 0 (WL#9631) – This work by Mayank Prasad changes the default of innodb_flush_neighbors from 1 (enable) to 0 (disable). This is done because fast IO (SSDs) is now the default for deployment. We expect that for the majority of users, this will result in a small performance gain. Users who are using slower hard drives may see a performance loss, and are encouraged to revert to the previous defaults by setting innodb_flush_neighbors=1.
- Change innodb_max_dirty_pages_pct_lwm default to 10 (WL#9630) – This work by Mayank Prasad changes the default of innodb_max_dirty_pages_pct_lwm from 0 (%) to 10 (%). With innodb_max_dirty_pages_pct_lwm=10, InnoDB will increase its flushing activity when >10% of the buffer pool contains modified (‘dirty’) pages. The motivation for this default change is to trade off peak throughput slightly, in exchange for more consistent performance. We do not expect the majority of users to see impact from this change, but symptomatically query throughput may be reduced after a number of sustained modifications. Users who wish to revert to the previous behavior can set innodb_max_dirty_pages_pct_lwm=0. The value of zero disables the increased flushing heuristic.
- Change innodb_max_dirty_pages_pct default to 90 (WL#9707) – This work by Mayank Prasad changes the default of innodb_max_dirty_pages_pct from 75 (%) to 90 (%).  With this change, InnoDB will allow a slightly greater number of modified (‘dirty’) pages in the buffer pool, at the risk of a lower amount of free-able space for other operations that require loading pages into the buffer pool. However in practice, InnoDB does not have the same reliance on innodb_max_dirty_pages_pct as it did in earlier versions of MySQL because of the introduction of a new low-watermark heuristic. With innodb_max_dirty_pages_pct_lwm, flushing activity increases at a much earlier point (default: 10%). Users wishing to revert to the previous behavior can set innodb_max_dirty_pages_pct=75 and innodb_max_dirty_pages_pct_lwm=0.
- Change default algorithm for calculating back_log (WL#9704) – This work by Abhishek Ranjan changes the algorithm used for the back_log default which is autosize (-1). The new algorithm is simply to set back_log equal to max_connections. Default value will be capped to maximum limit permitted by range of ‘back_log’ (65535). The old algorithm  was to set back_log = 50 + (max_connections / 5).
- Change max-allowed-packet compiled default to 64M (WL#8393) – This work by Abhishek Ranjan changes the default of max_allowed_packet from 4194304 (4M) to 67108864 (64M). The main advantage with this larger default is that fewer users receive errors about insert or query being larger than max_allowed_packet. Users wishing to revert to the previous behavior can set max_allowed_packet=4194304.
- Change max_error_count default to 1024 (WL#9686) – This work by Abhishek Ranjan changes the default of max_error_count from 64 to 1024. The effect is that MySQL will handle a larger number of warnings, e.g. for an UPDATE statement that touches 1000s of rows and many of them give conversion warnings (batched updates). There are no static allocations, so this change will only affect memory consumption for statements that generate lots of warnings.
- Enable event_scheduler by default (WL#9644) – This work by Abhishek Ranjan changes the default of event_scheduler from OFF to ON. This is seen as an enabler for new features in SYS, for example “kill idle transactions”.
- Enable binary log by default (WL#10470) – This work by Narendra Chauhan changes the default of –log-bin from OFF to ON. Nearly all production installations have the binary log enabled as it is used for replication and point-in-time recovery. Thus, by enabling binary log by default we eliminate one configuration step for users (enabling it later requires a mysqld restart). By enabling it by default we also get better test coverage and it becomes easier to spot performance regressions.
- Enable replication chains by default (WL#10479) – This work by Ganapati Sabhahit changes the default of log-slave-updates from OFF to ON.  This causes a slave to log replicated events into its own binary log. This option ensures correct behavior in various replication chain setups, which have become the norm today. This is also required for Group Replication.

### 默认值变更
默认情况下，根据系统资源自动调整InnoDB资源。引入了一个新的选项  innodb_dedicated_server，可以设置为OFF / ON（默认为OFF）。如果为ON，则下列InnoDB变量（如果未明确指定）的设置将相应缩为innodb_buffer_pool_size，innodb_log_file_size和innodb_flush_method。另请参阅Morgan Tocker的针对MySQL8.0创造性改善计划的博客文章。
- 将innodb_autoinc_lock_mode的默认值从连续的（1）更改为交错的（2）。因为现在默认的复制格式是基于行的复制，所以这个是可以实现的。但此更改与基于statement的复制不兼容，并且可能会破坏依赖于连续自增值一些应用程序或用户生成的测试套件。以前的默认值可以通过设置innodb_autoinc_lock_mode=1恢复;
- 将innodb_flush_neighbors的默认值从1（启用）更改为0（禁用）。这是因为现在大家默认部署服务都使用快速IO（SSD）。我们预计，对于大多数用户来说，这将导致较小的性能提升。如果使用较慢的硬盘驱动器性能可能会有一定损失，我们建议您通过设置innodb_flush_neighbors=1将其恢复为以前的默认值。
- 将innodb_max_dirty_pages_pct_lwm默认值从0(%)调整为10(%)。这样一来，当缓冲池中脏页超过10%的时候，InnoDB刷新脏页的效率会得到一定的提升。这个修改的动机是通过轻微的吞吐量的损失来换取来换取更连续平稳的性能。我们希望大部分用户不要太在意这个变更的影响，我们会通过不断地调整来缓解这个问题。对于想要恢复到之前状态的用户，可以设置innodb_max_dirty_pages_pct_lwm=0。0值禁止the increased flushing heuristic(这句不知道怎么翻译)
- 将innodb_max_dirty_pages_pct的默认值从75（％）更改为90（％）。通过这个更改，在存在需要将页面加载到缓冲池中的其他操作的可用空间较少的风险的情况下，InnoDB缓冲池中可以允许稍微更多的脏页存在。然而在实践中，因为引入了一种新的低水位启发式算法InnoDB与老版本中的innodb_max_dirty_pages_pct不一样。使用innodb_max_dirty_pages_pct_lwm，刷新脏页的点会被提升到一个更早的点（默认值：10％）。希望恢复以前配置的用户可以设置innodb_max_dirty_pages_pct=75和innodb_max_dirty_pages_pct_lwm=0。
- 修改了之前自动调整back_log的算法（-1）。新算法直接将back_log设置为等于max_connections。默认值将被限制到“back_log”（65535）范围允许的最大限制。旧的算法是设置的back_log = 50 + (max_connections / 5)。
- 将max_allowed_packet的默认值从4194304（4M）更改为67108864（64M）。这样的主要优势在于让更少的用户收到报错”插入或查询的时候大于max_allowed_packet”。想恢复到之前默认值的用户可以设置max_allowed_packet=4194304。
- 将max_error_count的默认值从64 更改为1024.这样MySQL可以处理更多的警告，例如，一个UPDATE语句触发 1000行的记录变更，这个过程会生成一些警告（批量更新）。没有静态分配，所以这个变更只会影响生成大量警告的语句的内存消耗。
- event_scheduler的默认值从OFF 更改为ON。这被视为对SYS中新功能的启用，例如“kill idle transaction”。
- log-bin的默认值从OFF更改为ON。因为复制和基于时间点恢复的需要，几乎所有的生产安装都启用了二进制日志。因此，通过默认启用二进制日志，免去了用户开启binlog之后需要重启mysqld服务的步骤。通过默认启用它，我们还可以获得更好的测试覆盖率，并且更容易发现性能回归。
- log-slave-updates的默认值从OFF改为ON。这样从机会把复制的事件记录到其自己的二进制日志中以确保在各种复制链中的参数设置正确，而这也已成为现在的一个标准，这还是组复制所必需的。


### Deprecation and Removal
- Remove query cache (WL#10824) – This work by Steinar Gunderson removes the query cache for 8.0. See also blog post Retiring Support for the Query Cache by Morgan Tocker. All related startup options and configuration variables are removed as well. HAVE_QUERY_CACHE will now return NO, so that well-behaved clients can check for this and behave accordingly. The SQL_NO_CACHE keyword will continue to exist, but will be ignored (no effect in the grammar). This is so that e.g. mysqldump can continue working.
- Rename tx_{read_only,isolation} variables to transaction_{read_only,isolation}  (WL#9636) – This work by Nisha Gopalakrishnan removes the system variables called tx_read_only and  tx_isolation. Use transaction_read_only and transaction_isolation instead. This is done to harmonize wording with command-line format –transaction_read_only and –transaction_isolation as well as with other transaction related system varaibles like transaction_alloc_block_size, transaction_allow_batching, and transaction_prealloc_size. See also Bug#70008 reported by Simon Mudd.
- Remove log_warnings option (WL#9676) – This work by Tatjana Nurnberg removes the old log-warnings option deprecated in 5.7. Use log_error_verbosity instead.
- Remove ignore_builtin_innodb option (WL#9675) – This work by Georgi Kodinov removes the old ignore_builtin_innodb options deprecated in 5.6. Even when used, these options have had no effect since MySQL 5.6.
- Remove ENCODE()/DECODE() functions (WL#10788) – This work by Georgi Kodinov removes the ENCODE() and DECODE() functions deprecated in 5.7.Use AES_ENCRYPT() and AES_DECRYPT() instead.
- Remove ENCRYPT(), DES_ENCRYPT(), and DES_DECRYPT() functions (WL#10789) –This work by Georgi Kodinov removes the ENCRYPT(), DES_ENCRYPT(),and DES_DECRYPT() functions deprecated in 5.7. Use AES_ENCRYPT() and AES_DECRYPT() instead.    
- Remove parameter secure_auth (WL#9674) – This work by Georgi Kodinov removes the secure_auth deprecated in 5.7. The option appears in server and clients. Even when used, these options have had no effect since MySQL 5.7. The secure-auth was used to control whether the mysql_old_password methods are allowed on the client and the server but this authentication method is now gone from both the client and the server.
- Remove EXPLAIN PARTITIONS and EXTENDED options (WL#9678) – This work by Sreeharsha Ramanavarapu removes the EXTENDED and PARTITIONS keywords from EXPLAINdeprecated in 5.7. Both EXTENDED and PARTITIONS output are enabled by default since 5.7, so these keywords are superfluous and thus removed.
- Remove unused date_format, datetime_format, time_format, max_tmp_tables (WL#9680) – This work by Sreeharsha Ramanavarapu removes system variables date_format, datetime_format, time_format,and max_tmp_tables. These variables have never been in use (or at least not been used in MySQL 4.1 or newer releases).
- Remove multi_range_count system variable (WL#10908) – This work by Sreeharsha Ramanavarapu removes the system variable multi_range_count deprecated in 5.1. Even when used, this option has had no effect since MySQL 5.5. From MySQL 5.5 and onwards, arbitrarily long lists of ranges can be processed.
- Remove the global scope of the sql_log_bin system variable (WL#10922) – This work by Luis Soares removes the global scope of the sql_log_bin system variable in MySQL 8.0. The sql_log_bin was set read only in MySQL 5.5, 5.6 and 5.7. In addition, reading this variable was deprecated in MySQL 5.7. See also Bug#67433 reported by Jeremy Cole.
- Deprecate master.info and relay-log.info files (WL#6959) – This work by Luis Soares implements a deprecation warning in the server when either relay-log-info-repository or master-info-repository are set to FILE instead of TABLE. The default setting is TABLE for both options and this is also the most crash-safe setup.
- Deprecate mysqlbinlog –stop-never-slave-server-id (WL#9633) – This work by Luis Soares implements a deprecation warning in the mysqlbinlog utility for the –stop-never-slave-server-id option. Use the –connection-server-id option instead.
- Deprecate mysqlbinlog- -short-form (WL#9632) – This work by Luis Soares implements a deprecation warning in the mysqlbinlog utility for the –short-form option. This option is not to be used in production (as stated in the docs) and is now too overloaded to be used even when testing.
- Deprecate IGNORE_SERVER_IDS when GTID_MODE=ON (WL#10963) – This work by Luis Soares implements a deprecation warning when users try to use CHANGE MASTER TOIGNORE_SERVER_IDS together with GTID_MODE=ON.  When GTID_MODE=ON, any transaction that has been applied is automatically filtered out, so there is no need for IGNORE_SERVER_IDS.
- Deprecate expire_logs_days (WL#10924) – This work by Neha Kumari adds a deprecation warning when users try to set expire_logs_days. Use the new variable binlog_expire_log_seconds instead. The new variable allows users to set expire time which need not be a multiple of days. This is the better way to set the expiration time and also more flexible, it makes the system variable expire_logs_days superfluous.

### 不建议使用以及被废除的特性
- 删除8.0的查询缓存功能，其所有相关的启动选项和配置变量也被删除。为了客户端可以检查并执行相应的操作，现在HAVE_QUERY_CACHE变量将返回NO。SQL_NO_CACHE关键字将继续存在，但将被忽略（在语法中不起作用）。也就是说，例如mysqldump这种工具可以继续正常使用。
- 将tx_{read_only,isolation}变量重命名为transaction_{read_only,isolation}这样做是为了与命令行格式的-transaction_read_only和-transaction_isolation以及与其它事务相关的系统变量像transaction_alloc_block_size，transaction_allow_batching和transaction_prealloc_size互相统一。
- 删除了5.7中不推荐使用的旧日志警告选项。改用log_error_verbosity。
- 删除了在5.6中不推荐使用的旧的ignore_builtin_innodb选项。从MySQL 5.6起，即使使用，些选项也没有任何效果。
- 删除了5.7中不推荐使用的ENCODE（）和DECODE（）函数。请改用AES_ENCRYPT（）和AES_DECRYPT（）。
- 删除了5.7中不推荐使用的ENCRYPT（），DES_ENCRYPT （）和DES_DECRYPT（）函数。请改用AES_ENCRYPT（）和AES_DECRYPT（）。
- 删除了5.7中不推荐使用的secure_auth。该配置用在服务端和客户端上。从MySQL5.7开始，即使使用这选项也没有效果。之前secure-auth是用来控制mysql_old_password的方法是否允许在客户端和服务端上使用，但这个身份验证方法现在都没用了。
- 删除了在5.7中弃用的EXPLAIN和PARTITIONS关键字。从5.7开始，EXTENDED和PARTITIONS输出都被默认启用，所以这些关键字是多余的，因此被删除。
- 删除系统变量date_format，datetime_format，time_format和max_tmp_tables。这些变量从来没有被使用过（或至少没有在MySQL 4.1或更新的版本中使用）。
- 删除了5.1中不推荐使用的系统变量multi_range_count。即使使用，从MySQL5.5开始，可以处理任意长的范围列表，所以这个变量没有任何效果。**(这个没怎么理解这个参数)**
- MySQL 8.0中删除了全局系统变量sql_log_bin。sql_log_bin在MySQL 5.5，5.6和5.7被设置为只读。另外，在MySQL5.7中这个变量已经不可读。另见Jeremy Cole报道的Bug＃67433。(这块有问题，我5.7实例可以看到这个参数)
- 在服务端实现了当relay-log-info-repository或master-info-repository设置为FILE而不是表时会发出不推荐告警。默认设置是两个选项的TABLE，这也是最安全的设置。
- 在使用-stop-never-slave-server-id参数的mysqlbinlog应用中实现了一个不推荐的警告。请改用-connection-server-id参数。
- 在-short-form参数的mysqlbinlog应用中实现了一个废弃警告。此选项不会在生产中使用（如文档中所述），并且现在过载甚至在测试时也不会被使用。
- 实现了一项弃用警告。当GTID_MODE = ON时，已应用的任何事务都会自动过滤掉，因此不需要IGNORE_SERVER_IDS。
- 在用户尝试设置expire_logs_days时添加了一项弃用警告。让改用新的变量binlog_expire_log_seconds。新变量允许用户设置到期时间**(这里没太理解，两个效果不一样吗)**。这是设置到期时间的更好方法，也更灵活，这使得系统变量expire_logs_logs_days变得有些多余

## That’s it for now. Thank you for using MySQL!
## 这就是本次所有亮点，感谢您对MySQL的支持！
