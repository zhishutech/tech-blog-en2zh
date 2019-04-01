> 原文 https://lefred.be/content/replace-mariadb-10-3-by-mysql-8-0/
>
> 作者：[lefred](https://lefred.be/content/author/lefred/)
>
> 翻译：无名队

# 为什么要迁移到MySQL8.0？

*MySQL 8.0 brings a lot of new features. These features make MySQL database much more secure (like new authentication, secure password policies and management, …) and fault tolerant (new data dictionary), more powerful (new redo log design, less contention, extreme scale out of InnoDB, …), better operation management (SQL Roles, instant add columns), many (but really many!) replication enhancements and native group replication… and finally many cool stuff like the new Document Store, the new MySQL Shell and MySQL InnoDB Cluster that you should already know if you follow this blog (see these [TOP 10 for features for developers](https://lefred.be/content/top-10-mysql-8-0-features-for-developers/) and this [TOP 10 for DBAs & OPS](https://lefred.be/content/top-10-mysql-8-0-features-for-dbas-ops/)).*

MySQL8.0带来了很多新特性。这些新特性使得MySQL数据库更加安全（例如新的认证方式，安全的密码策略和管理方式，...）和容错（新的数据字典）功能更强大（新的redo设计，争用更少，极度扩展InnoDB，…），更好的操作管理（SQL角色，即时添加列 ），很多（其实真的很多）复制增强和本地组复制...最后还有很多很酷的东西例如新的文档，全新的MySQL Shell和MySQL InnoDB cluster，如果你follow了以下这些博客的话你应该已经知道了（[TOP 10 for features for developers](https://lefred.be/content/top-10-mysql-8-0-features-for-developers/) 和[TOP 10 for DBAs & OPS](https://lefred.be/content/top-10-mysql-8-0-features-for-dbas-ops/))）

## 不再是替代品

*We saw in this previous post how to migrate from MariaDB 5.5 (default on CentOS/RedHat 7) to MySQL. This was a straight forward migration as at the time MariaDB was a drop in replacement for MySQL…**but this is not the case anymore since MariaDB 10.x !***

我们在上一篇文章中看到了如何从MariaDB 5.5（在CentOS/RedHat7上默认）迁移到MySQL。这是一个直接的迁移，因为当时MariDB是MySQL的替代品…但是从MariaDB 10.x开始情况就不一样了。

让我们开始迁移到MySQL8.0

## 选项

*Two possibilities are available to us:*

1. *Use logical dump for schemes and data*
2. *Use logical dump for schemes and transportable InnoDB tablespaces for the data*

我们有两种方式：

- 对schema和数据逻辑导出
- 对schema逻辑导出，使用InnoDB表空间交换导出数据

## 准备迁移

**方式1-全部逻辑导出**

*It’s recommended to avoid to have to deal with `mysql.*` tables are they won’t be compatible, I recommend you to save all that information and import the required entries like users manually. It’s maybe the best time to do some cleanup.*

这是推荐的方式以避免处理`mysql.*`表，因为它们不兼容，我建议你保存所有的信息并且手动导入需要的条目例如用户表。这可能是做一些清理的最佳时机。

*As we are still using our WordPress site to illustrate this migration. I will dump the `wp` database:*

我们仍然使用我们的WordPress网络来说明这种迁移。我将导出`wp`数据库：

```mysql
mysqldump -B wp> wp.sql
```

> *MariaDB doesn’t provide* `mysqlpump`*, so I used the good old* `mysqldump`*. There was a nice article this morning about MySQL logical dump solutions,* [see it here](https://mydbops.wordpress.com/2019/03/26/mysqldump%E2%80%8B-vs-mysqlpump-vs-mydumper/)*.*
>
> MariaDB没有提供`mysqlpump`，所以我们使用了旧的`mysqldump`。这里有一篇很好的关于MySQL逻辑导出解决方案的文章，[请看这里](https://mydbops.wordpress.com/2019/03/26/mysqldump%E2%80%8B-vs-mysqlpump-vs-mydumper/)

**方式2**-表结构导出 & InnoDB表传输

*First we take a dump of our database without the data (`-d`):*

首先我们导出数据库结构不带数据[-d]

```mysql
mysqldump -d -B wp > wp_nodata.sq
```

*Then we export the first table space:*

然后我们导出第一个表空间

```mysql
[wp]> flush tables wp_comments for export;
Query OK, 0 rows affected (0.008 sec
```

*We copy it to the desired location (the `.ibd` and the `.cfg`):*

我们将其拷贝到所需的位置(`.ibd`和`.cfg`)

```shell
cp wp/wp_comments.ibd ~/wp_innodb/
cp wp/wp_comments.cfg ~/wp_innodb/
```

*And finally we unlock the table:*

最后，我们解锁表

```mysql
[wp]> unlock tables;
```

*These operation above need to be repeated for all the tables ! If you have a large amount of table I encourage you to script all these operations.*

以上这些操作需要为每个表都重复做一次！如果你有很多表，我建议你使用脚本来做这些操作

## 替换二进制文件/安装MySQL 8.0

*Unlike previous version, if we install MySQL from the Community Repo as seen on this post, MySQL 8.0 won’t be seen as a conflicting replacement for MariaDB 10.x. To avoid any conflict and installation failure, we will replace the MariaDB packages by the MySQL ones using the **swap**command of `yum`:*

与以前的版本不同，如果我们从社区网站上安装MySQL，MySQL8.0将不会被视为MariaDB 10.x兼容替代品。为了避免任何不兼容和安装失败，我们将使用`yum swap`的命令来将MySQL包替换MariaDB的包

```shell
yum swap -- install mysql-community-server mysql-community-libs-compat -- \ 
remove MariaDB-server MariaDB-client MariaDB-common MariaDB-compat
```

> *This new yum command is very useful, and allow other dependencies like php-mysql or postfix for example to stay installed without breaking some dependencies*
>
> 这个新的yum命令非常有用，并且允许其他依赖项（如php-mysql或postfix）保持安装而不会破坏某些依赖项

*The result of the command will be something similar to:*

这个命令的结果类似于

```shell
Removed:
   MariaDB-client.x86_64 0:10.3.13-1.el7.centos            
   MariaDB-common.x86_64 0:10.3.13-1.el7.centos            
   MariaDB-compat.x86_64 0:10.3.13-1.el7.centos           
   MariaDB-server.x86_64 0:10.3.13-1.el7.centos           
 Installed:
   mysql-community-libs-compat.x86_64 0:8.0.15-1.el7
   mysql-community-server.x86_64 0:8.0.15-1.el7                                     
 Dependency Installed:
   mysql-community-client.x86_64 0:8.0.15-1.el7
   mysql-community-common.x86_64 0:8.0.15-1.el7
   mysql-community-libs.x86_64 0:8.0.15-1.el7
```

*Now the best is to empty the datadir and start `mysqld`:*

现在最好清空datadir然后启动`mysqld`：

```shell
rm -rf /var/lib/mysql/*
systemctl start mysql
```

*This will start the initialize process and start MySQL.*

*As you may know, by default MySQL is now more secure and a new password has been generated to the `root` user. You can find it in the error log (`/var/log/mysqld.log`):*

这将会开始初始化进程然后启动MySQL

你可能知道，默认情况下，MySQL现在更加安全，并且已为`root`用户生成密码。你可以在错误日志(/var/log/mysqld.log)中找到它：

```mysql
2019-03-26T12:32:14.475236Z 5 [Note] [MY-010454] [Server] 
A temporary password is generated for root@localhost: S/vfafkpD9a
```

*At first login with the `root` user, the password must be changed:*

第一次使用`root`用户登录，必须更改密码：

```mysql
mysql -u root -p
mysql> set password='Complicate1#'
```

## 添加凭据

*Now we need to create our database (`wp`), our user and its credentials.*

现在我们需要创建我们的数据库（wp），我们的用户及其凭据

> Please, note that the PHP version used by default in CentOS might now be yet compatible with the new default secure authentication plugin, therefor we will have to create our user with the older authentication plugin, `mysql_native_password`. For more info see these posts:
>
> 请注意，CentOS中默认使用的PHP版本现在可能与新的默认安全认证插件兼容，因此我们必须使用旧的认证插件创建我们的用户`mysql_native_password`。有关更多信息，请参阅以下帖子
>
> \- [在不破坏旧应用程序的情况下迁移到MySQL 8.0](https://lefred.be/content/migrating-to-mysql-8-0-without-breaking-old-application/)
>
> \- [Drupal和MySQL 8.0.11 - 我们在那里吗？](https://lefred.be/content/drupal-and-mysql-8-0-11-are-we-there-yet/)
>
> \- [Joomla！和MySQL 8.0.12](https://lefred.be/content/joomla-and-mysql-8-0-12/)
>
> \- [PHP 7.2.8和MySQL 8.0](https://lefred.be/content/php-7-2-8-mysql-8-0/)

```mysql
mysql> create user 'wp'@'127.0.0.1' identified with 
       'mysql_native_password' by 'fred';
```

> by default, this password (*fred*) won’t be allowed with the default [password policy](https://dev.mysql.com/doc/refman/8.0/en/validate-password-options-variables.html#sysvar_validate_password.policy).
>
> To not have to change our application, it’s possible to override the policy like this:
>
> 默认情况下，这个密码(fred)不会被默认的密码策略通过。为了不修改我们的程序，可以通过这样来覆盖策略：
>
> ```mysql
> mysql> set global validate_password.policy=LOW;
> mysql> set global validate_password.length=4
> ```

*It’s possible to see the user and its authentication plugin easily using the following query:*

可以通过如下sql很轻松地查看用户及相应的认证插件

```mysql
mysql> select Host, User, plugin,authentication_string from mysql.user where User='wp';
 +-----------+------+-----------------------+-------------------------------------------+
 | Host      | User | plugin                | authentication_string                     |
 +-----------+------+-----------------------+-------------------------------------------+
 | 127.0.0.1 | wp   | mysql_native_password | *6C69D17939B2C1D04E17A96F9B29B284832979B7 |
 +-----------+------+-----------------------+-------------------------------------------+
```

*We can now create the database and grant the privileges to our user:*

现在我们可以创建数据库并授权给我们的用户:

```mysql
mysql> create database wp;
Query OK, 1 row affected (0.00 sec)
mysql> grant all privileges on wp.* to 'wp'@'127.0.0.1';
Query OK, 0 rows affected (0.01 sec)
```

## 恢复数据

*This process is also defined by the options chosen earlier.*

此过程也由前面的选择而定

方式1

*This option, is the most straight forward, one restore and our site is back online:*

这个方式最直接，一次还原然后我们的网站重新上线：

```mysql
mysql -u wp -pfred wp <~/wp.sql
```

方式2

*This operation is more complicated as it requires more steps.*

*First we will have to restore all the schema with no data:*

这个方式相对来说更复杂因为它需要更多步骤

首先我们需要先恢复schema结构

```mysql
mysql -u wp -pfred wp <~/wp_nodata.sql
```

*And now for every tables we need to perform the following operations:*

然后，对于每张表我们需要进行如下操作

```mysql
mysql> alter table wp_posts discard tablespace;

cp ~/wp_innodb/wp_posts.ibd /var/lib/mysql/wp/
cp ~/wp_innodb/wp_posts.cfg /var/lib/mysql/wp/
chown mysql. /var/lib/mysql/wp/wp_posts.*

mysql> alter table wp_posts import tablespace
```

*Yes, this is required for all tables, this is why I encourage you to script it if you choose this option.*

是的，所有的表都需要这么操作，所以这也是为什么我建议你使用脚本来跑如果你选择了这种方式

## 结论

*So as you could see, it’s still possible to migrate from MariaDB to MySQL but since 10.x, this is not a drop in replacement anymore and requires several steps including logical backup.*

正如你看到的，仍然可以从MariaDB迁移到MySQL，但是从10.x开始，这不再是替代品，需要几个步骤，包括逻辑备份。