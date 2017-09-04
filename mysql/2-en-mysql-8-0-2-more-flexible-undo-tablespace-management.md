原文：[MySQL 8.0.2 More Flexible Undo Tablespace Management](http://mysqlserverteam.com/mysql-8-0-2-more-flexible-undo-tablespace-management/)
作者：[Kevin Lewis](http://mysqlserverteam.com/author/kevin/)

In MySQL 8.0.2 DMR we will introduce features which make managing undo tablespaces easier in InnoDB.

The main improvement is that you can now create and drop undo tablespaces at any time.  You can change the config file setting before any startup, whether recovery is needed or not.  And you can either increase or decrease the number of undo tablespaces while the engine is busy.

**innodb_undo_tablespaces:**  Undo Tablespaces contain Rollback Segments which in turn contain undo logs.  Undo Logs are used to ‘rollback’ transactions and to create earlier versions of data which is used by Multi Version Concurrency Control to present a consistent image of the database during a transaction.

Previously, the number of undo tablespaces that InnoDB uses was established when the database was initialized.  It can now be set to any value between 0 and 127 at any time; at startup in either the config file or on the command line, or while online by issuing ‘SET GLOBAL INNODB_UNDO_TABLESPACES=n’.

When you choose zero undo tablespaces, all rollback segments are tracked by the system tablespace.  This is the old way of storing Rollback Segments before separate Undo Tablespaces were added in version 5.6.  We are trying to move away from using the system tablespace in this way, so the default value is not set to 2.  In the near future, the minimum value will become 2, which means that the system tablespace will not be used for any rollback segments. So please do not keep innodb_undo_tablespaces=0 in your config files.

**innodb_undo_log_truncate:**   We chose a minimum of 2 undo tablespaces because you need at least 2 in order for one of them to be truncated.  Undo truncation allows InnoDB to shrink the undo tablespace size after unusually large transactions. Previously, the innodb_undo_log_truncate setting defaulted to OFF.  With version 8.0.2 it defaults to ON.

**innodb_rollback_segments:** This can now be set to any value between 1 and 128 at any time; at startup in either the config file or the command line, or while online by issuing ‘SET GLOBAL INNODB_ROLLBACK_SEGMENTS=n’.

This setting used to be the number of rollback segments that the whole server could support.  It is now the number of rollback segments in each undo tablespace, allowing a greater number of rollback segments to be used by concurrent transactions.  The default value is still 128.

**innodb_undo_logs:**  This setting was introduced in 5.6 as an alternate or alias of innodb_rollback_segments. It was a little confusing in terminology since in InnoDB, ‘Undo Logs’ are stored in Rollback Segments, which are file segments of an Undo Tablespace.  In v8.0.2, we are dropping the use of this setting and requiring Innodb_rollback_segments to be used instead.  The latest released version 5.7.19 contains deprecation warnings if it is used.

**Undo Tablespace Name and Location:** Undo tablespaces are located in the directory specified by the setting innodb_undo_directory. If that setting is not used, they are created in the ‘datadir’ location.  Previously they had names like ‘undo001’, ‘undo002’, etc. In v8.0.2 DMR they have names like ‘undo_001’, ‘undo_002’, etc. The reason for the name change is that these newer undo tablespaces contain a new header page that maps the locations of each rollback segment it contains.   In version 5.6 when separate undo tablespaces were introduced, their rollback segment header page numbers were tracked in the system tablespace which limited the number of rollback segments for the whole instance to 128.

Since each undo tablespace can now track its own rollback segments with this new page, these are really new types of undo tablespaces and need to have a different naming convention.  This is also the reason that innodb_rollback_segments now defines the number of rollback segments per undo tablespace instead of the number for the whole MySQL instance.

**Automatic Upgrade:** Before this change, the system tablespace tracked all rollback segments whether they were in the system tablespace or in undo tablespaces.  If you start MySQL 8.0.2 on an existing database that uses the system tablespace to track rollback segments, at least 2 new undo tablespaces will be generated automatically.  This will be common since the previous default value for innodb_undo_tablespaces was 0.  Mysql 5.7 databases will go through an upgrade process which among other things will create the new DD from the old FRM files.  As part of this process, at least 2 new undo tablespaces will be created as well.  InnoDB can still use the existing rollback segments and undo tablespaces defined in the system tablespace if they have undo logs in them at startup.  But it will not assign these old rollback segments to any new transactions.  So once undo recovery is finished and these undo logs are not needed anymore, the old undo tablespaces are deleted.

**Advantages and Benefits:**  This change allows you to dynamically add more undo tablespaces and rollback segments as a database installation grows.  With more undo tablespaces, it is easier to use undo tablespace truncation to minimize the disk space dedicated to rollback segments.  Also, more rollback segments mean that concurrent transactions are more likely to use separate rollback segments for their undo logs which results in less contention for the same resources.

Thanks for using MySQL!

