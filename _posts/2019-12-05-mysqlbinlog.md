---
layout: post
title: "mysqlbinlog使用"
date: 2019-12-05 23:05:30 +0800
catalog: ture
multilingual: false
tags:
    - mysql
---

### mysqlbinlog介绍

​		在mysql中，任意时间对数据库所做的修改，都会被记录到日志文件中。例如，当你添加了一个新的表，或者更新了一条数据，这些事件都会被存储到二进制日志文件中。二进制日志文件在MySQL主从复合中是非常有用的，主服务器会发送其数据到远程服务器中。

​		mysqlbinlog 命令，以用户可视的方式展示出二进制日志中的内容。同时，也可以将其中的内容读取出来，供其他MySQL实用程序使用。

**mysql开启binlog**

```bash
[mysqld]
log-bin = mysql-bin
binlog_format = ROW
expire-logs-days = 3
```

### mysql binlog相关命令

#### 常用命令

```bash
mysql> show binary logs;
mysql> show binlog events;  # 不指定显示最新的binlog文件
mysql> show binlog events in 'mysql-bin.000002' limit 3,5\G  #从第三行开始列出5行

mysqlbinlog  --start-date="2019-10-15 16:30:00" --stop-date="2019-10-15 17:00:00" mysql_bin.000001 > /data/1.sql 

mysqlbinglog常见的选项有以下几个：
--start-datetime
从二进制日志中读取指定时间戳或者本地计算机时间之后的日志事件。
--stop-datetime
从二进制日志中读取指定时间戳或者本地计算机时间之前的日志事件。
--start-position     
从二进制日志中读取指定position 事件位置作为开始。
--stop-position
从二进制日志中读取指定position 事件位置作为事件截至
# 例子
mysqlbinlog /usr/local/mysql/data/mysql-bin.000001 > /opt/mysql-bin.000001.sql
mysqlbinlog --stop-position=287 /usr/local/mysql/data/mysql-bin.000002 > /opt/287.sql
mysqlbinlog --start-position=416 /usr/local/mysql/data/mysql-bin.000002 > /opt/416.sql
```

#### binlog_format  ROW

```bash
mysqlbinlog -v -v --database=test --base64-output=DECODE-ROWS --start-datetime="2019-08-09 10:42:36" --stop-datetime="2019-08-10 10:42:36" mysql-bin.008508
```

#### 指定数据库

```bash
mysqlbinlog -d crm mysql-bin.000001
```



### 相关问题

#### unknown variable

```bash
# /usr/local/mysql/bin/mysqlbinlog  --help
/usr/local/mysql/bin/mysqlbinlog: unknown variable 'default-character-set=utf8mb4'
```

原因是mysqlbinlog这个工具无法识别binlog中的配置中的default-character-set=utf8这个指令。

可使用  **mysqlbinlog --no-defaults**

### 参考

[MySQL数据库的灾难备份与恢复](https://blog.51cto.com/xiaorenwutest/1941848)

[超级有用的15个mysqlbinlog命令](http://www.ttlsa.com/mysql/super-useful-mysqlbinlog-command/)


