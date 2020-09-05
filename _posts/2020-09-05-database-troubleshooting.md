---
layout: post
title: "database troubleshooting"
date: 2020-09-05 23:03:30 +0800
catalog: ture
multilingual: false
tags:
    - database
---



## mysql

[TOC]

### 向mysql导入数据失败（MySQL server has gone away）

mysql日志中报错`[Warning] Aborted connection 5 to db: 'iamstigris' user: 'root' host: 'localhost' (Got a packet bigger than 'max_allowed_packet' bytes)`



```sql
mysql>**show VARIABLES like '%max_allowed_packet%';**

显示：
+--------------------------+---------------------------------+
 | Variable_name            | Value                       |
 +--------------------------+---------------------------------+
 | max_allowed_packet       | 4194304            |
 | slave_max_allowed_packet | 1073741824 |
 +--------------------------+----------------------------------+
 2 rows in set
```

（1）显示：**主**最大允许包（max_allowed_packet）等于4M，**从**最大允许包（slave_max_allowed_packet）等于1G；

`max_allowed_packet` 针对的是一个事务中的一行记录大小，当一行记录超过了限制的大小，将会报错。sql文件中每次insert完进同一张表的所有数据被称为一个数据包（packet），max_allowed_packet就是来限制这个的大小的阈值，大于这个值，mysql的I/O连接会关闭，就会报这个错。

[Got a packet bigger than 'max_allowed_packet' bytes](https://blog.csdn.net/superit401/article/details/77480078)

```bash
# 命令行设置
set global max_allowed_packet = 2*1024*1024*10 # 20M
```



### ERROR 1839 (HY000) at line 24: @@GLOBAL.GTID_PURGED can only be set when @@GLOBAL.GTID_MODE = ON.



恢复mysqldump备份的数据库报错,可以在sql文件中注释掉 

也可以加-f

```bash
 nohup  /usr/local/mysql/bin/mysql -f -uroot -pssss-HGmHFDHyyM feather < /data1/feather_1008.sql > /dev/null 2>&1 &
[1] 107160
```



### Error: Couldn't read status information for table innodb_index_stats ()
mysqldump: Couldn't execute 'show create table `innodb_index_stats`': Table 'mysql.innodb_index_stats' doesn't exist (1146)

mysqldump发生报错

在日志中查看,mysql数据库下以下表都损坏

```
-rw-rw---- 1 mysql mysql  11K Jul 27  2017 innodb_index.stats.frm
-rw-rw---- 1 mysql mysql  11K Jul 27  2017 innodb_table_stats.frm
-rw-rw---- 1 mysql mysql  11K Jul 27  2017 slave_master_info.frm
-rw-rw---- 1 mysql mysql 9.2K Jul 27  2017 slave_relay_log_info.frm
-rw-rw---- 1 mysql mysql 9.1K Jul 27  2017 slave_worker_info.frm
```

在$MYSQL_HOME/share/mysql_system_tables.sql，search到建表语句 

在datadir中删除损坏的文件（上面的那五个）,重启mysql，重建那写表。

建表语句可以在其它数据库中show create table tablename;



参考：[InnoDB: Error: Table "mysql"."innodb_table_stats" not found.](https://blog.csdn.net/mchdba/article/details/25109427)

### ERROR 1794 (HY000): Slave is not configured or failed to initialize properly. You must at least set --server-id to enable either a master or a slave. Additional error messages can be found in the MySQL error log.

测试发现以下几个系统表不存在.

```
innodb_table_stats
innodb_index_stats
slave_master_info
slave_relay_log_info
slave_worker_info
```

解决的方法.

```bash
1.删除上述系统表
drop table mysql.innodb_index_stats;
drop table mysql.innodb_table_stats;
drop table mysql.slave_master_info;
drop table mysql.slave_relay_log_info;
drop table mysql.slave_worker_info;



2.删除相关的.frm .ibd文件
rm -f innodb_index_stats*
rm -f innodb_table_stats*
rm -f slave_master_info*
rm -f slave_relay_log_info*
rm -f slave_worker_info*



3.重新创建上述系统表
```

```sql
CREATE TABLE `innodb_index_stats` (
  `database_name` varchar(64) COLLATE utf8_bin NOT NULL,
  `table_name` varchar(64) COLLATE utf8_bin NOT NULL,
  `index_name` varchar(64) COLLATE utf8_bin NOT NULL,
  `last_update` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `stat_name` varchar(64) COLLATE utf8_bin NOT NULL,
  `stat_value` bigint(20) unsigned NOT NULL,
  `sample_size` bigint(20) unsigned DEFAULT NULL,
  `stat_description` varchar(1024) COLLATE utf8_bin NOT NULL,
  PRIMARY KEY (`database_name`,`table_name`,`index_name`,`stat_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin STATS_PERSISTENT=0;

CREATE TABLE `innodb_table_stats` (
  `database_name` varchar(64) COLLATE utf8_bin NOT NULL,
  `table_name` varchar(64) COLLATE utf8_bin NOT NULL,
  `last_update` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `n_rows` bigint(20) unsigned NOT NULL,
  `clustered_index_size` bigint(20) unsigned NOT NULL,
  `sum_of_other_index_sizes` bigint(20) unsigned NOT NULL,
  PRIMARY KEY (`database_name`,`table_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin STATS_PERSISTENT=0;

CREATE TABLE `slave_master_info` (
  `Number_of_lines` int(10) unsigned NOT NULL COMMENT 'Number of lines in the file.',
  `Master_log_name` text CHARACTER SET utf8 COLLATE utf8_bin NOT NULL COMMENT 'The name of the master binary log currently being read from the master.',
  `Master_log_pos` bigint(20) unsigned NOT NULL COMMENT 'The master log position of the last read event.',
  `Host` char(64) CHARACTER SET utf8 COLLATE utf8_bin NOT NULL DEFAULT '' COMMENT 'The host name of the master.',
  `User_name` text CHARACTER SET utf8 COLLATE utf8_bin COMMENT 'The user name used to connect to the master.',
  `User_password` text CHARACTER SET utf8 COLLATE utf8_bin COMMENT 'The password used to connect to the master.',
  `Port` int(10) unsigned NOT NULL COMMENT 'The network port used to connect to the master.',
  `Connect_retry` int(10) unsigned NOT NULL COMMENT 'The period (in seconds) that the slave will wait before trying to reconnect to the master.',
  `Enabled_ssl` tinyint(1) NOT NULL COMMENT 'Indicates whether the server supports SSL connections.',
  `Ssl_ca` text CHARACTER SET utf8 COLLATE utf8_bin COMMENT 'The file used for the Certificate Authority (CA) certificate.',
  `Ssl_capath` text CHARACTER SET utf8 COLLATE utf8_bin COMMENT 'The path to the Certificate Authority (CA) certificates.',
  `Ssl_cert` text CHARACTER SET utf8 COLLATE utf8_bin COMMENT 'The name of the SSL certificate file.',
  `Ssl_cipher` text CHARACTER SET utf8 COLLATE utf8_bin COMMENT 'The name of the cipher in use for the SSL connection.',
  `Ssl_key` text CHARACTER SET utf8 COLLATE utf8_bin COMMENT 'The name of the SSL key file.',
  `Ssl_verify_server_cert` tinyint(1) NOT NULL COMMENT 'Whether to verify the server certificate.',
  `Heartbeat` float NOT NULL,
  `Bind` text CHARACTER SET utf8 COLLATE utf8_bin COMMENT 'Displays which interface is employed when connecting to the MySQL server',
  `Ignored_server_ids` text CHARACTER SET utf8 COLLATE utf8_bin COMMENT 'The number of server IDs to be ignored, followed by the actual server IDs',
  `Uuid` text CHARACTER SET utf8 COLLATE utf8_bin COMMENT 'The master server uuid.',
  `Retry_count` bigint(20) unsigned NOT NULL COMMENT 'Number of reconnect attempts, to the master, before giving up.',
  `Ssl_crl` text CHARACTER SET utf8 COLLATE utf8_bin COMMENT 'The file used for the Certificate Revocation List (CRL)',
  `Ssl_crlpath` text CHARACTER SET utf8 COLLATE utf8_bin COMMENT 'The path used for Certificate Revocation List (CRL) files',
  `Enabled_auto_position` tinyint(1) NOT NULL COMMENT 'Indicates whether GTIDs will be used to retrieve events from the master.',
  PRIMARY KEY (`Host`,`Port`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 STATS_PERSISTENT=0 COMMENT='Master Information';

CREATE TABLE `slave_relay_log_info` (
  `Number_of_lines` int(10) unsigned NOT NULL COMMENT 'Number of lines in the file or rows in the table. Used to version table definitions.',
  `Relay_log_name` text CHARACTER SET utf8 COLLATE utf8_bin NOT NULL COMMENT 'The name of the current relay log file.',
  `Relay_log_pos` bigint(20) unsigned NOT NULL COMMENT 'The relay log position of the last executed event.',
  `Master_log_name` text CHARACTER SET utf8 COLLATE utf8_bin NOT NULL COMMENT 'The name of the master binary log file from which the events in the relay log file were read.',
  `Master_log_pos` bigint(20) unsigned NOT NULL COMMENT 'The master log position of the last executed event.',
  `Sql_delay` int(11) NOT NULL COMMENT 'The number of seconds that the slave must lag behind the master.',
  `Number_of_workers` int(10) unsigned NOT NULL,
  `Id` int(10) unsigned NOT NULL COMMENT 'Internal Id that uniquely identifies this record.',
  PRIMARY KEY (`Id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 STATS_PERSISTENT=0 COMMENT='Relay Log Information';

CREATE TABLE `slave_worker_info` (
  `Id` int(10) unsigned NOT NULL,
  `Relay_log_name` text CHARACTER SET utf8 COLLATE utf8_bin NOT NULL,
  `Relay_log_pos` bigint(20) unsigned NOT NULL,
  `Master_log_name` text CHARACTER SET utf8 COLLATE utf8_bin NOT NULL,
  `Master_log_pos` bigint(20) unsigned NOT NULL,
  `Checkpoint_relay_log_name` text CHARACTER SET utf8 COLLATE utf8_bin NOT NULL,
  `Checkpoint_relay_log_pos` bigint(20) unsigned NOT NULL,
  `Checkpoint_master_log_name` text CHARACTER SET utf8 COLLATE utf8_bin NOT NULL,
  `Checkpoint_master_log_pos` bigint(20) unsigned NOT NULL,
  `Checkpoint_seqno` int(10) unsigned NOT NULL,
  `Checkpoint_group_size` int(10) unsigned NOT NULL,
  `Checkpoint_group_bitmap` blob NOT NULL,
  PRIMARY KEY (`Id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 STATS_PERSISTENT=0 COMMENT='Worker Information';
```

4.重启数据库



### 一次mysql 5.5升级到5.6导致的ERROR 1805

主从错误信息  Last_SQL_Errno: 1805

```sql
Worker 0 failed executing transaction '' at master log mysql-bin.000063, end_log_pos 1007288192; Error 'Column count of mysql.user is wrong. Expected 43, found 42. The table is probably corrupted' on query. Default database: ''. Query: 'grant replication slave on *.* to 'repl'@'10.80.139.193' identified by '5R23YNJxrQzetZLg''
```





前阵子将mysql数据库由5.5.14升级到5.6.36，升级后所有的业务数据都正常。运行了几天后，发现在主库上添加用户失败，错误提示为：ERROR 1805 (HY000): Column count of mysql.user is wrong，提示mysql.user表列的数目不对。还真是个坑。下面是其解决方案。

```sql
从上面的描述来看，说列的数目不对，因此我们升级后环境mysql.user表的建表语句
(robin@localhost)[(none)]>show variables like 'version';
+---------------+------------+
| Variable_name | Value |
+---------------+------------+
| version | 5.5.14-log |
+---------------+------------+
1 row in set (0.01 sec)

(robin@localhost)[(none)]>show create table mysql.user\G
*************************** 1. row ***************************
Table: user
Create Table: CREATE TABLE `user` (
`Host` char(60) COLLATE utf8_bin NOT NULL DEFAULT '',
............
`plugin` char(64) COLLATE utf8_bin DEFAULT '',
`authentication_string` text COLLATE utf8_bin,
PRIMARY KEY (`Host`,`User`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='Users and global privileges'

安装一个新的mysql 5.6.36后查看mysql.user表的建表语句
mysql> show variables like 'version';
+---------------+--------+
| Variable_name | Value |
+---------------+--------+
| version | 5.6.36 |
+---------------+--------+
1 row in set (0.01 sec)

mysql> show create table mysql.user\G
*************************** 1. row ***************************
Table: user
Create Table: CREATE TABLE `user` (
...........
`plugin` char(64) COLLATE utf8_bin DEFAULT 'mysql_native_password',
`authentication_string` text COLLATE utf8_bin,
`password_expired` enum('N','Y') CHARACTER SET utf8 NOT NULL DEFAULT 'N',
PRIMARY KEY (`Host`,`User`)   ---上行为多出的列，即先前使用主从复制时password_expired列没有
) ENGINE=MyISAM DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='Users and global privileges'

--为升级后的mysql 5.6.36手工添加这个password_expired列
mysql> alter table mysql.user add column `password_expired` enum('N','Y') CHARACTER SET utf8 NOT NULL DEFAULT 'N';
Query OK, 10 rows affected (0.00 sec)
Records: 10 Duplicates: 0 Warnings: 0

--再次创建用户成功
mysql> grant all privileges on *.* to 'henry'@'%' identified by 'henrypwd';
Query OK, 0 rows affected (0.00 sec)
--------------------- 
原文：https://blog.csdn.net/leshami/article/details/78521217 
```

### Relay log write failure: could not queue event from master

**show slave status\G**中查看 **Exec_Master_Log_Pos**的位置，去master上相应的binlog日志中查找下一个正确的位置，然后**change master**改变正确的位置。

```sql
stop slave;
reset slave;
change master to master_host='10.90.9.110',master_user='repl',master_password='5R23YNrQzetZLg',master_log_file='mysql-bin.039927',master_log_pos=0;
start slave;
show slave status\G
```

### 不能启动

删掉 mysql-bin.index

### Cannot execute the current event group in the parallel mode.

Cannot execute the current event group in the parallel mode. Encountered event Update_rows, relay-log name /data/site/mysqldata/mysql-relay-bin.001689, position 354 which prevents execution of this event group in parallel mode. Reason: the event is a part of a group that is unsupported in the parallel execution mode.

```bash
stop slave;
SET GLOBAL slave_parallel_workers=0;
start slave;

正常后
SET GLOBAL slave_parallel_workers=4;
```



### [MySQL error: The maximum column size is 767 bytes](https://stackoverflow.com/questions/30761867/mysql-error-the-maximum-column-size-is-767-bytes)

```bash
set global innodb_file_format = BARRACUDA;
set global innodb_large_prefix = ON;
create table test (........) ENGINE=InnoDB DEFAULT CHARSET=utf8 ROW_FORMAT=DYNAMIC;
```



[MySQL数据库运维之主从复制延迟问题排查](https://segmentfault.com/a/1190000015311988)

[MySQL innodb_table_stats表不存在的解决方法](https://blog.51cto.com/10742668/1957314)

## redis

[TOC]

### redis-cluster主从切换后，造成主从不同步

**从库报错日志**

```json
22804:S 22 Feb 13:26:39.269 # Connection with master lost.
22804:S 22 Feb 13:26:39.269 * Caching the disconnected master state.
22804:S 22 Feb 13:26:39.453 * Connecting to MASTER 10.80.97.147:6380
22804:S 22 Feb 13:26:39.453 * MASTER <-> SLAVE sync started
22804:S 22 Feb 13:26:39.453 * Non blocking connect for SYNC fired the event.
22804:S 22 Feb 13:26:39.454 * Master replied to PING, replication can continue...
22804:S 22 Feb 13:26:39.454 * Trying a partial resynchronization (request 4fcc29abe35fa7458ae69ee4a89a49c408f2f33b:2511885719192).
22804:S 22 Feb 13:29:20.358 * Full resync from master: 4fcc29abe35fa7458ae69ee4a89a49c408f2f33b:2512010876867
22804:S 22 Feb 13:29:20.358 * Discarding previously cached master state.
22804:S 22 Feb 13:33:10.840 * MASTER <-> SLAVE sync: receiving 8161106002 bytes from master
22804:S 22 Feb 13:34:05.563 * MASTER <-> SLAVE sync: Flushing old data
22804:S 22 Feb 13:36:39.114 * MASTER <-> SLAVE sync: Loading DB in memory
22804:S 22 Feb 13:39:45.539 * MASTER <-> SLAVE sync: Finished with success
22804:S 22 Feb 13:39:45.664 # Connection with master lost.
```

看到有很多的`Connection with master lost`的错误，然后slave尝试要求master进行部分同步：`Trying a partial resynchronization (request 4fcc29abe35fa7458ae69ee4a89a49c408f2f33b:2511885719192)`。但是master还是要求全量同步：`Full resync from master: 4fcc29abe35fa7458ae69ee4a89a49c408f2f33b:2512010876867`。然后就是全量同步的过程。上面的过程每几分钟就发生一次，一直重复着。

**主库报错日志**

```json
25021:M 22 Feb 13:26:39.454 * Slave 10.80.91.147:6380 asks for synchronization
25021:M 22 Feb 13:26:39.454 * Unable to partial resync with slave 10.80.91.147:6380 for lack of backlog (Slave request was: 2511885719192).
25021:M 22 Feb 13:26:39.454 * Can't attach the slave to the current BGSAVE. Waiting for next BGSAVE for SYNC
13351:C 22 Feb 13:29:14.960 * DB saved on disk
13351:C 22 Feb 13:29:15.940 * RDB: 9362 MB of memory used by copy-on-write
25021:M 22 Feb 13:29:18.859 * Background saving terminated with success
25021:M 22 Feb 13:29:18.859 * Starting BGSAVE for SYNC with target: disk
25021:M 22 Feb 13:29:20.358 * Background saving started by pid 13956
13956:C 22 Feb 13:33:07.109 * DB saved on disk
13956:C 22 Feb 13:33:08.087 * RDB: 9396 MB of memory used by copy-on-write
25021:M 22 Feb 13:33:10.839 * Background saving terminated with success
25021:M 22 Feb 13:34:04.057 * Synchronization with slave 10.80.91.147:6380 succeeded
25021:M 22 Feb 13:34:11.121 * 10000 changes in 60 seconds. Saving...
25021:M 22 Feb 13:34:12.635 * Background saving started by pid 14533
25021:M 22 Feb 13:34:23.959 * FAIL message received from 7dc97cd72cf74483287692ab111c887a7c8a94e7 about e6cd89fdcb009cc10dfc8a08d4ecfcaa3ec8086a
25021:M 22 Feb 13:36:39.214 * Clear FAIL state for node e6cd89fdcb009cc10dfc8a08d4ecfcaa3ec8086a: slave is reachable again.
25021:M 22 Feb 13:37:38.022 # Client id=219273 addr=10.80.91.147:56276 fd=54 name= age=659 idle=2 flags=S db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=0 obl=0 oll=4742 o
mem=77774182 events=rw cmd=psync scheduled to be closed ASAP for overcoming of output buffer limits.
25021:M 22 Feb 13:37:38.027 # Connection with slave 10.80.91.147:6380 lost.
```

master日志中发现：`scheduled to be closed ASAP for overcoming of output buffer limits.`

这是由于`client-output-buffer-limit`限制太小造成的。

redis的默认配置为：

```bash
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
```

**问题处理**

**增大client-output-buffer-limit限制**

	修改配置为`config set client-output-buffer-limit "slave 536870912 536870912 60"`(512mb 512mb 60)从库还是有问题 。

**拷贝主库rbd文件到从库**

主库执行bgsave，产生新的rbd文件，传到从库，重新启动从库。从库初始化后，主从同步状态正常。

正常后取消了`client-output-buffer-limit`的限制：

```
config set client-output-buffer-limit "slave 0 0 0"
config set client-output-buffer-limit "pubsub 0 0 0"
```

















 



	从master日志我们可以知道为什么主库不能对从库进行部分同步：`Unable to partial resync with slave 10.80.91.147:6380 for lack of backlog (Slave request was: 2511885719192)`。这个涉及到Redis的部分同步的实现。在2.8版本，redis使用了新的复制方式，引入了replication backlog以支持部分同步。

> The master then starts background saving, and starts to buffer all new commands received that will modify the dataset. When the background saving is complete, the master transfers the database file to the slave, which saves it on disk, and then loads it into memory. The master will then send to the slave all buffered commands. This is done as a stream of commands and is in the same format of the Redis protocol itself.



	当主服务器进行命令传播的时候，maser不仅将所有的数据更新命令发送到所有slave的replication buffer，还会写入replication backlog。当断开的slave重新连接上master的时候，slave将会发送psync命令（包含复制的偏移量offset），请求partial resync。如果请求的offset不存在，那么执行全量的sync操作，相当于重新建立主从复制。

replication backlog是一个环形缓冲区，整个master进程中只会存在一个，所有的slave公用。backlog的大小通过repl-backlog-size参数设置，默认大小是1M，其大小可以根据每秒产生的命令、（master执行rdb bgsave） +（master发送rdb到slave） + （slave load rdb文件）时间之和来估算积压缓冲区的大小，repl-backlog-size值不小于这两者的乘积。

所以在主从同步的时候，slave会落后master的时间 ＝（master执行rdb bgsave）+ (master发送rdb到slave) + (slave load rdb文件) 的时间之和。 然后如果在这个时间master的数据变更非常巨大，超过了replication backlog，那么老的数据变更命令就会被丢弃，导致需要全量同步。

#### client-output-buffer-limit





#### 处理办法

**命令行**

```bash
config set client-output-buffer-limit "slave 0 0 0"
config set client-output-buffer-limit "pubsub 0 0 0"
```

**配置文件**

```bash
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 0 0 0
client-output-buffer-limit pubsub 0 0 0
```



#### 参考

https://blog.csdn.net/rongge2008/article/details/53503526