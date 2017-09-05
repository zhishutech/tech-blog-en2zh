原文： [MySQL 8.0.2 More Flexible Undo Tablespace Management](http://mysqlserverteam.com/mysql-8-0-2-more-flexible-undo-tablespace-management/)

作者：[Kevin Lewis](http://mysqlserverteam.com/author/kevin/)

翻译团队：天一阁

In MySQL 8.0.2 DMR we will introduce features which make managing undo tablespaces easier in InnoDB.

  在MySQL 8.0.2 DMR版本中，我们将介绍在InnoDB中可更易管理的UNDO表空间的特性。

  The main improvement is that you can now create and drop undo tablespaces at any time. You can change the config file setting before any startup, whether recovery is needed or not. And you can either increase or decrease the number of undo tablespaces while the engine is busy.

  主要的改进是，你现在可以自由地创建或删除UNDO表空间。你可以在启动前更改配置文件设置，无论是否需要恢复。当引擎处于繁忙状态时，你可以增加或减少UNDO表空间的数量。

**innodb_undo_tablespaces**: Undo Tablespaces contain Rollback Segments which in turn contain undo logs. Undo Logs are used to ‘rollback’ transactions and to create earlier versions of data which is used by Multi Version Concurrency Control to present a consistent image of the database during a transaction.

  **innodb_undo_tablespaces**: UNDO表空间包括回滚段，而回滚段又包括UNDO日志。UNDO日志用于'回滚'事务和创建MVCC所需要的早期版本的数据，以便在一个事务中保证数据库快照的一致性。

  Previously, the number of undo tablespaces that InnoDB uses was established when the database was initialized. It can now be set to any value between 0 and 127 at any time; at startup in either the config file or on the command line, or while online by issuing ‘SET GLOBAL INNODB_UNDO_TABLESPACES=n’.

  之前的版本，当数据库初始化时，InnoDB所使用的UNDO表空间的数量已建立。现在可以随时将其值设置为0~127之间的任意值，可通过启动时读取的配置文件，或者命令行，或者通过在线‘SET GLOBAL INNODB_UNDO_TABLESPACES=n’.

  When you choose zero undo tablespaces, all rollback segments are tracked by the system tablespace. This is the old way of storing Rollback Segments before separate Undo Tablespaces were added in version 5.6. We are trying to move away from using the system tablespace in this way, so the default value is not set to 2. In the near future, the minimum value will become 2, which means that the system tablespace will not be used for any rollback segments. So please do not keep innodb_undo_tablespaces=0 in your config files.

  当你选择不使用独立UNDO表空间时，所有的回滚段被系统表空间所跟踪。这是在 5.6版本添加独立UNDO表空间之前存储回滚段的老办法。我们尝试通过这种方式远离使用系统表空间，所以默认值不会设置为2。在不久的将来最小值将变成2，这表明系统表空间将不会被用作任何回滚段。所以请不要在你的配置文件中设置innodb_undo_tablespaces=0。

**innodb_undo_log_truncate**: We chose a minimum of 2 undo tablespaces because you need at least 2 in order for one of them to be truncated. Undo truncation allows InnoDB to shrink the undo tablespace size after unusually large transactions. Previously, the innodb_undo_log_truncate setting defaulted to OFF. With version 8.0.2 it defaults to ON.

  **innodb_undo_log_truncate**：我们选择'2'作为UNDO表空间的最小值，因为你至少需要两个，以便其中一个被truncated。UNDO清除操作允许InnoDB在异常大事务之后缩小UNDO表空间的大小。以前，innodb_undo_log_truncate的默认值为OFF。8.0.2版本该默认值为ON。

**innodb_rollback_segments**: This can now be set to any value between 1 and 128 at any time; at startup in either the config file or the command line, or while online by issuing ‘SET GLOBAL INNODB_ROLLBACK_SEGMENTS=n’.

  **innodb_rollback_segments**: 选择可以随时设置为1~128之间的任何值。可通过启动时读取的配置文件，或者命令行，或者通过在线‘SET GLOBAL INNODB_ROLLBACK_SEGMENTS=n’.

  This setting used to be the number of rollback segments that the whole server could support. It is now the number of rollback segments in each undo tablespace, allowing a greater number of rollback segments to be used by concurrent transactions. The default value is still 128.

  这个选项曾是用于整个服务器可以支持的回滚段数。现在为每一个UNDO表空间的回滚段数，允许并发事务使用更多的回滚段数，该选项默认值仍为128。

**innodb_undo_logs**: This setting was introduced in 5.6 as an alternate or alias of innodb_rollback_segments. It was a little confusing in terminology since in InnoDB, ‘Undo Logs’ are stored in Rollback Segments, which are file segments of an Undo Tablespace. In v8.0.2, we are dropping the use of this setting and requiring Innodb_rollback_segments to be used instead. The latest released version 5.7.19 contains deprecation warnings if it is used.

  **innodb_undo_logs**：该选项在5.6中作为innodb_rollback_segments的替代或者别名所引入。在InnoDB中术语有一点儿混乱，‘Undo Logs’被存储在回滚段，这是UNDO表空间的文件段。在8.0.2版本，我们正在弃用该选项并且要求使用Innodb_rollback_segments来替代。在最新的发布的5.7.19版本包含了若使用则抛出弃用warnings。

  Undo Tablespace Name and Location: Undo tablespaces are located in the directory specified by the setting innodb_undo_directory. If that setting is not used, they are created in the ‘datadir’ location. Previously they had names like ‘undo001’, ‘undo002’, etc. In v8.0.2 DMR they have names like ‘undo_001’, ‘undo_002’, etc. The reason for the name change is that these newer undo tablespaces contain a new header page that maps the locations of each rollback segment it contains. In version 5.6 when separate undo tablespaces were introduced, their rollback segment header page numbers were tracked in the system tablespace which limited the number of rollback segments for the whole instance to 128.

  UNDO表空间命名和位置：UNDO表空间位于innodb_undo_directory所指定的目录中。如果该选项没有被使用，则被创建于‘datadir’中。以前，他们被命名为‘undo001’, ‘undo002’。在8.0.2 DMR版本，他们被称作‘undo_001’, ‘undo_002’等。改名的原因是在新的UNDO表空间中包含了一个新的头页面，其映射了每一个回滚段的位置。在5.6版本中，当独立UNDO表空间被引入时，其回滚段头页面号由系统表空间所跟踪，并且限制整个实例的回滚段数为128。

  Since each undo tablespace can now track its own rollback segments with this new page, these are really new types of undo tablespaces and need to have a different naming convention. This is also the reason that innodb_rollback_segments now defines the number of rollback segments per undo tablespace instead of the number for the whole MySQL instance.

  由于每个UNDO表空间可以使用这个新页来跟踪自己的回滚段，这些是真正新的UNDO表空间类型，并需要有不同的命名约定。这也是innodb_rollback_segments现在定义每个UNDO表空间回滚段数量而不是整个MySQL实例数量的原因。

**Automatic Upgrade**: Before this change, the system tablespace tracked all rollback segments whether they were in the system tablespace or in undo tablespaces. If you start MySQL 8.0.2 on an existing database that uses the system tablespace to track rollback segments, at least 2 new undo tablespaces will be generated automatically. This will be common since the previous default value for innodb_undo_tablespaces was 0. Mysql 5.7 databases will go through an upgrade process which among other things will create the new DD from the old FRM files. As part of this process, at least 2 new undo tablespaces will be created as well. InnoDB can still use the existing rollback segments and undo tablespaces defined in the system tablespace if they have undo logs in them at startup. But it will not assign these old rollback segments to any new transactions. So once undo recovery is finished and these undo logs are not needed anymore, the old undo tablespaces are deleted.

**Automatic Upgrade**：在此更变之前，系统表空间会跟踪所有回滚段，无论他们位于系统表空间还是UNDO表空间。如果现在在一个使用系统表空间跟踪回滚段的已经存在了的数据库上启动MySQL 8.0.2。将自动生成至少两个新的UNDO表空间。这将是很正常的，因为innodb_undo_tablespaces之前的默认值为0。MySQL 5.7数据库将直接通过升级进程，其中包括从旧的FRM文件创建新的DD。作为此进程的一部分，还将创建至少两个新的UNDO表空间。在启动时，InnoDB在系统表空间中定义的UNDO 日志，仍可以使用现有的回滚段和UNDO表空间。但是它不会将这些老的回滚段分配给任何新的事物。所以一旦UNDO恢复完成，并不再需要这些UNDO日志，旧的UNDO表空间也将被删除。

**Advantages and Benefits**: This change allows you to dynamically add more undo tablespaces and rollback segments as a database installation grows. With more undo tablespaces, it is easier to use undo tablespace truncation to minimize the disk space dedicated to rollback segments. Also, more rollback segments mean that concurrent transactions are more likely to use separate rollback segments for their undo logs which results in less contention for the same resources.

**Advantages and Benefits**：此更改允许你在数据库规模增长时，动态地添加更多的UNDO表空间和回滚段。使用更多的UNDO表空间，可以更加简易地使用UNDO表空间清除来最小化用于存放回滚段的磁盘空间。此外，更多的回滚段意味着并发事务可尽可能的使用单独的回滚段，以减少相同资源的争用。

Thanks for using MySQL!
