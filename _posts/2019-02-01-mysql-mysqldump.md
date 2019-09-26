---
layout: post
title: "mysql使用mysqldump进行备份"
date: 2019-02-1 15:00:34 +0800
catalog: ture  
multilingual: false
tags: 
    - mysql
---


> mysqldump备份是逻辑备份，数据量小时可以使用，数据量大推荐使用xtrabackup.

### mysqldump参数

-A 备份所有   **--all-databases**

-B 备份多个库，并添加use 库名；create database 库等的功能。 **--databases**

通过mysqldump备份的文件，如果用了`--all-databases`或`--databases`选项，则在备份文件中包含CREATE DATABASE和USE语句，故并**不需要指定一个数据库名去恢复备份文件。

#### --master-data  和  --single-transaction

mysqldump导出数据时， `--master-data `当这个参数的值为1的时候，mysqldump出来的文件就会包括CHANGE MASTER TO这个语句，CHANGE MASTER TO后面紧接着就是file和position的记录，在slave上导入数据时就会执行这个语句，salve就会根据指定这个文件位置从master端复制binlog。默认情况下这个值是1 。

当这个值是2的时候，chang master to也是会写到dump文件里面去的，但是这个语句是被注释的状态。


`--single-transaction`
InnoDB 表在备份时，通常启用选项 --single-transaction 来保证备份的一致性，实际上它的工作原理是设定本次会话的隔离级别为：REPEATABLE READ，以确保本次会话(dump)时，不会看到其他会话已经提交了的数据。

### 一些备份命令

**导出整个数据库结构和数据**

`mysqldump -h localhost -uroot -p123456 database --master-data=2 --single-transaction > dump.sql`

 

**导出单个数据表结构和数据**

`mysqldump -h localhost -uroot -p123456  database table --master-data=2 --single-transaction > dump.sql`

 

**导出整个数据库结构（不包含数据)**

`mysqldump -h localhost -uroot -p123456  -d database > dump.sql`

 

**导出单个数据表结构（不包含数据）**

`mysqldump -h localhost -uroot -p123456  -d database table > dump.sql`

### 备份脚本(Innodb)

```bash
#!/bin/bash
username='backup'
password='password'
dbtime=$(date +%a)
backup_time=$(date +"%Y-%m-%d %H:%M:%S")
backup_dir="/data/backup/mysql"
backup_log="/data/backup/mysql/mysql_backup.log"

mkdir -p ${backup_dir}
/usr/local/mysql/bin/mysqldump --all-databases --master-data=2 --single-transaction -u${username} -p${password} -h127.0.0.1 |gzip > ${backup_dir}/databases_${dbtime}.sql.gz &&\
echo "${backup_time}: all databases backup success" >> /var/log/mysql_backup.log || echo "${backup_time}: all databases backup error" >> ${backup_log}

for dbname in `/usr/local/mysql/bin/mysql -u${username} -p${password} -h127.0.0.1 -e "show databases"|sed '1,2d'`
do
    if [ $dbname != 'performance_schema' ] && [ $dbname != 'test' ];then
    /usr/local/mysql/bin/mysqldump ${dbname} --master-data=2 --single-transaction -u${username} -p${password} -h127.0.0.1 |gzip > ${backup_dir}/${dbname}_${dbtime}.sql.gz &&\
    echo "${backup_time}: ${dbname} backup success" >> /var/log/mysql_backup.log || echo "${backup_time}: ${dbname} backup error" >> ${backup_log}
    fi
done

#30 1 * * * /data/script/cron/mysqlbak.sh > /dev/null 2>&1
```

### 还原
`mysql -uroot -plisai <BackupName.sql`
