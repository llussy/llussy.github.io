---
layout: post
title: "mysql使用xtrabackup进行备份"
date: 2019-01-30 14:38:34 +0800
catalog: ture  
multilingual: false
tags: 
    - mysql
---


> **xtrabackup是开源的MySQL备份工具，物理备份，效率很不错。**



### xtrabackup安装

```bash
yum install perl-Digest-MD5 -y
yum install http://www.percona.com/downloads/percona-release/redhat/0.1-4/percona-release-0.1-4.noarch.rpm
 yum install percona-xtrabackup-24 -y
或者
yum install -y http://10.80.21.156/rpm/all/xtrabackup-2.1.6-1.x86_64.rpm
```

### 备份

创建备份用户

```bash
####参考 percona官网 https://www.percona.com/doc/percona-xtrabackup/2.4/using_xtrabackup/privileges.html
mysql> CREATE USER 'bkpuser'@'localhost' IDENTIFIED BY 'xxxxxx';
mysql> GRANT RELOAD, LOCK TABLES, PROCESS, REPLICATION CLIENT ON *.* TO 'bkpuser'@'localhost';
mysql> FLUSH PRIVILEGES;
```

innobackupex在备份过程中，会给非innodb表上读锁，会给innodb表上元数据信息锁。

**--defaults-file在第一个参数**

#### 全量备份命令

```bash
#不压缩备份
innobackupex --defaults-file=/etc/my.cnf --user=bkpuser --password=xxxxxx  /data/backup/full/

#tar压缩备份
/data/bin/xtrabackup/bin/innobackupex  --user=bkpuser --password=xxxxxx --defaults-file=/etc/my.cnf --stream=tar $fulldir| gzip - > $fulldir/bakcup_$DATE.tar.gz
```

#### 备份单个表

```bash
/data/bin/xtrabackup/bin/innobackupex  --user=bkpuser --password=xxxxxx --defaults-file=/etc/my.cnf --include='panel.y_article_text_base' --stream=tar $fulldir| gzip - > $fulldir/y_article_text_base_$DATE.tar.gz
```



#### 备份脚本

```bash
# 定时任务
00 02 * * * /bin/sh /data/bin/xtrabak.sh --cron   >>/tmp/xtrabak.log 2>&1
# cat /data/bin/xtrabak.sh
#!/bin/sh
PATH=/sbin:/bin:/usr/sbin:/usr/bin:/usr/local/mysql/bin:/root/bin:/data/bin/xtrabackup/bin
export PATH
fulldir="/data/backup/full"
DATE=`date +%F`
LDIR="/data/backup/full"
cd /data/bin/xtrabackup/bin
mkdir -p /data/backup/full
/data/bin/xtrabackup/bin/innobackupex  --user=bkpuser --password=xxxxxx --defaults-file=/etc/my.cnf  $fulldir >>/tmp/xtraback.log 
doc=`ls -l /data/backup/full/ |grep ^d |awk '{print $9}'|tail -1`
cd /data/backup/full
tar -zcvf $doc.tar.gz $doc
#rm -rf /data/backup/full/$doc
/usr/bin/rsync -avzP --bwlimit=10240 /data/backup/full/$doc.tar.gz 10.80.20.156::lisai/xxxxx
find $fulldir  -maxdepth 1 -type f -mtime +5 -exec echo "removeing:"  {} \;|xargs rm -rf
find $fulldir  -maxdepth 1 -type d -mtime +1 -exec echo "removeing:"  {} \;|xargs rm -rf
```

#### 备份mysql5.7命令

```bash
/data/bin/xtrabackup/bin/innobackupex --defaults-file=/etc/my.cnf --user=root --password=xxxxxxx -S /usr/local/mysql/mysql.sock /data/backup/full
```

### 还原

```bash
##mysql datadir要为空  /data/site/mysqldata   /etc/init.d/mysqld stop
innobackupex --apply-log /home/lisai/2018-06-26_16-44-54/
innobackupex --defaults-file=/etc/my.cnf --copy-back  /home/lisai/2018-06-26_16-44-54/

### my.cnf中要指定datadir
chown -R mysql:mysql /data/site/mysqldata/
启动mysql /etc/init.d/mysqld start

#主从同步
# more 2018-06-26_16-44-54/xtrabackup_binlog_info
mysql-bin.000013        909     08273eb0-4dd5-11e8-990e-005056bd1bdc:1-119

mysql> change master to master_host = '10.80.97.151',master_port =3306,master_user = 'repl',master_password = 'xxxxxx',MASTER_LOG_FILE='mysql-bin.000010', MASTER_LOG_POS=15091987,master_connect_retry=30;

mysql> start slave;

```

**遇到的错误**

```bash
#主从都开启了gtid，在设置从库的时候遇到了问题
#ERROR 1776 (HY000): Parameters MASTER_LOG_FILE, MASTER_LOG_POS, RELAY_LOG_FILE and RELAY_LOG_POS cannot be set when MASTER_AUTO_POSITION is active
解决：
mysql>  change master to master_auto_position=0;
Query OK, 0 rows affected (0.34 sec)


 
# ERROR 1840 (HY000) at line 24: GTID_PURGED can only be set when GTID_EXECUTED is empty.
解决：
mysql> reset master

```

### 单表备份恢复

**备份单个表**

`--include='panel.y_article_text_history'`

```bash
#!/bin/sh
PATH=/sbin:/bin:/usr/sbin:/usr/bin:/usr/local/mysql/bin:/root/bin:/data/bin/xtrabackup/bin
export PATH
fulldir="/data/backup/full"
DATE=`date +%F`
LDIR="/data/backup/full"
cd /data/bin/xtrabackup/bin
/data/bin/xtrabackup/bin/innobackupex  --user=bkpuser --password=xxxxxx --defaults-file=/etc/my.cnf --include='panel.y_article_text_history' --stream=tar $fulldir| gzip - > $fulldir/y_article_text_history_$DATE.tar.gz
```

**导出表**

```bash
cd /data/backup/mysql
tar -ixf  y_article_text_history_2018-10-31.tar.gz -C y_article_text_history/
innobackupex --apply-log --export /data/backup/mysql/y_article_text_history/
```

**还原表**

**主要步骤**   `定义表–删除表空间–拷贝*.ibd/*.cfg文件–导入表空间`

show create table xxxxx;可以查看创建表的命令

```sql
CREATE TABLE `y_article_text_history` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `create_time` date DEFAULT NULL,
  `news_id` varchar(255) NOT NULL,
  `title` varchar(255) DEFAULT NULL,
  `content` mediumtext COMMENT '去除字段换行和回车符 : REPLACE(REPLACE(t.content, CHAR(10), ''''), CHAR(13),'''')',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=22520932 DEFAULT CHARSET=utf8 COMMENT='文章正文表';
```



```bash
mysql > ALTER TABLE sentiment.y_article_text_history DISCARD TABLESPACE;
# cp /data/backup/mysql/y_article_text_history/panel/{y_article_text_history.ibd,y_article_text_history.cfg} /data/site/mysqldata/panel/
# chown -R mysql.mysql /data/site/mysqldata/
mysql > ALTER TABLE sentiment.y_article_text_history IMPORT TABLESPACE;
```

**参考**

[利用Percona XtraBackup进行单表备份恢复](http://www.simlinux.com/2014/09/13/percona-xtrabackup-info.html)



### xtrabackup备份从库  恢复为主库的从库

**备份时加上--slave-info**

```bash
/data/bin/xtrabackup/bin/innobackupex  --user=bkpuser --password=s3cret --defaults-file=/etc/my.cnf --slave-info /data/backup/full >>/tmp/xtraback.log 
```

**恢复**

```bash
innobackupex --apply-log /data/backup/full/2018-11-01_17-50-46/
innobackupex --defaults-file=/etc/my.cnf --copy-back /data/backup/full/2018-11-01_17-50-46/
chown -R mysql:mysql /data/site/mysqldata/
/etc/init.d/mysqld start
先 reset slave;
```



```bash

cat xtrabackup_slave_info 
SET GLOBAL gtid_purged='99e0a235-4f54-11e7-9d7e-6c0b84af252f:1-191315666:191315668-517069640, c9c570ff-8e40-11e7-b7cc-6c0b84af23cf:1-9';
CHANGE MASTER TO MASTER_AUTO_POSITION=1

mysql> SET GLOBAL gtid_purged='08f59d96-c23a-11e8-900b-005056aa9f48:5-53, d44b665e-c239-11e8-900a-005056aaf654:1-9';
Query OK, 0 rows affected (0.02 sec)

mysql> CHANGE MASTER TO MASTER_AUTO_POSITION=1;
```



### 可能出现的问题

#### 报错信息：Got fatal error 1236 from master when reading data from binary log

```bash
 Got fatal error 1236 from master when reading data from binary log: 'The slave is connecting using CHANGE MASTER TO MASTER_AUTO_POSITION = 1, but the master has purged binary logs containing GTIDs that the slave requires.'
```

解决方法：

```bash
master:
master > show global variables like 'GTID_EXECUTED';
+---------------+-------------------------------------------+
| Variable_name | Value                                     |
+---------------+-------------------------------------------+
| gtid_executed | 9a511b7b-7059-11e2-9a24-08002762b8af:1-14 |
+---------------+-------------------------------------------+

slave> set global GTID_EXECUTED="9a511b7b-7059-11e2-9a24-08002762b8af:1-14"
ERROR 1238 (HY000): Variable 'gtid_executed' is a read only variable

slave> set global GTID_EXECUTED="9a511b7b-7059-11e2-9a24-08002762b8af:1-14"
ERROR 1238 (HY000): Variable 'gtid_executed' is a read only variable

错误！记住，我们从master获取GTID_EXECUTED，并且在slave上设置为GTID_PURGED。

slave> set global GTID_PURGED="9a511b7b-7059-11e2-9a24-08002762b8af:1-14";
ERROR 1840 (HY000): GTID_PURGED can only be set when GTID_EXECUTED is empty.

再次出错，GTID_EXECUTED在手动更改GTID_PURGED之前应为空，但我们无法使用SET更改它，因为它是一个只读变量。更改它的唯一方法是使用reset master（是的，在从属服务器上）：


slave1> reset master;
slave1 > show global variables like 'GTID_EXECUTED';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| gtid_executed |       |
+---------------+-------+
slave1 > set global GTID_PURGED="9a511b7b-7059-11e2-9a24-08002762b8af:1-14";
slave1> start slave io_thread;
slave1> show slave statusG
[...]
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
[...]

```



参考： [How to create/restore a slave using GTID replication in MySQL 5.6](https://www.percona.com/blog/2013/02/08/how-to-createrestore-a-slave-using-gtid-replication-in-mysql-5-6/)



> 后续完善增量备份

