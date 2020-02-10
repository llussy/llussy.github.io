---
layout: post
title: "mysql summary"
date: 2020-02-10 12:40:30 +0800
catalog: ture
multilingual: false
tags:
    - mysql
---

[toc]

### mysql常用操作

#### 初始化

```bash
/usr/local/mysql/scripts/mysql_install_db --defaults-file=/etc/my.cnf --basedir=/usr/local/mysql/ --datadir=/data/mysqldata --user=mysql

# 更新root密码
update mysql.user set password=password('xxxxxxx') where user='root';
flush privileges;

# 连接其他端口
mysql -uroot -p -h127.0.0.1 -P3307
```

#### 查看变量

```sql
show global variables like "%read_only%";
set global read_only=1; #1是只读，0是读写

show global variables like "%max_allowed_packet%";

show global variables like "%server_id%";
```

#### 数据表相关

```sql
创建
CREATE DATABASE `blog` CHARACTER SET utf8 COLLATE utf8_general_ci;

删除
drop table y_sentiment_grab_site;

清空表数据，保留表结构
truncate table mds_ifengad_ids_report_ids_20180427;

表的行数
select COUNT(*)  from y_user_info_base;
```

#### 慢查询

```sql
# 命令行关闭慢日志 改好权限后 再开启

mysql> set global slow_query_log='OFF';
Query OK, 0 rows affected (0.05 sec)

mysql> set global slow_query_log='ON';
Query OK, 0 rows affected (2.38 sec)


show variables like 'slow_query%';
set global long_query_time=10;  #单位s


```

#### 锁相关

```sql
show engine innodb status;

select * from information_schema.innodb_trx;
select * from information_schema.innodb_locks;
select * from information_schema.innodb_lock_waits;
```

#### 查看数据库相关信息

查看数据库大小

```sql
use information_schema;
select concat(round(sum(data_length/1024/1024),2),'MB') as data from tables;  #所有数据库的大小

select concat(round(sum(data_length/1024/1024),2),'MB') as data from tables where table_schema='database_name';  #某个数据库的大小
```

查看数据库某个表的大小

```sql
use information_schema;
select concat(round(sum(data_length/1024/1024),2),'MB') as data from tables where table_schema='database_name' and table_name='table_name';
```

查看数据库表的行数

```sql
use information_schema;
select table_name,table_rows from tables where TABLE_SCHEMA = 'database-name' order by table_rows desc;
```

#### 导入数据库单个表

```bash
./mysql -uroot -p --database iamstigris --table iams_site < /data/backup/mysql/iamstigris_Tue.sql
```

### mysql权限

#### 存储过程权限

在mysql存储过程出现的同时，用户权限也增加了5种，其中和存储过程有关的权限有 三种：
**ALTER ROUTINE 编辑或删除存储过程**
**CREATE ROUTINE 建立存储过程**
**EXECUTE 运行存储过程**
在使用GRANT创建用户的时候分配这三种权限。 存储过程在运行的时候默认是使用建立者的权限运行的。

```bash
select `name` from mysql.proc where db = 'xx' and `type` = 'PROCEDURE' ;        #存储过程
select `name` from mysql.proc where db = 'xx' and `type` = 'FUNCTION'   ;       #函数

show procedure status;                                                           #存储过程
show function status;                                                            #函数

SELECT * from information_schema.VIEWS   #视图
SELECT * from information_schema.TABLES  #表
SELECT * FROM triggers T WHERE trigger_name=”mytrigger” #触发器


# grant 创建、修改、删除 MySQL 数据表结构权限。
grant create on testdb.* to developer@'192.168.0.%';
grant alter on testdb.* to developer@'192.168.0.%';
grant drop on testdb.* to developer@'192.168.0.%';

# grant 操作 MySQL 外键权限。
grant references on testdb.* to developer@'192.168.0.%';

# grant 操作 MySQL 临时表权限。
grant create temporary tables on testdb.* to developer@'192.168.0.%';

#grant 操作 MySQL 索引权限。
grant index on testdb.* to developer@'192.168.0.%';

# grant 操作 MySQL 视图、查看视图源代码 权限。
grant create view on testdb.* to developer@'192.168.0.%';
grant show view on testdb.* to developer@'192.168.0.%';

# grant 操作 MySQL 存储过程、函数 权限。
grant create routine on testdb.* to developer@'192.168.0.%'; -- now, can show procedure status
grant alter routine on testdb.* to developer@'192.168.0.%'; -- now, you can drop a procedure
grant execute on testdb.* to developer@'192.168.0.%';


# grant 作用在存储过程、函数上
grant execute on procedure testdb.pr_add to 'dba'@'localhost'
grant execute on function testdb.fn_add to 'dba'@'localhost'
```

### 索引简单操作

#### 普通索引

```bash
# 创建索引
CREATE INDEX indexName ON mytable(username(length));  
如果是CHAR，VARCHAR类型，length可以小于字段实际长度；如果是BLOB和TEXT类型，必须指定 length。

# 修改表结构
ALTER table tableName ADD INDEX indexName(columnName);

# 创建表时创建索引
CREATE TABLE mytable(  
ID INT NOT NULL,
username VARCHAR(16) NOT NULL,  
INDEX [indexName] (username(length))
);  

# 删除
DROP INDEX [indexName] ON mytable;
```

#### 唯一索引

**索引列的值必须唯一，但允许有空值。如果是组合索引，则列值的组合必须唯一。**

```bash
# 创建索引
CREATE UNIQUE INDEX indexName ON mytable(username(length))

# 修改表结构
ALTER table mytable ADD UNIQUE [indexName] (username(length))

# 创建表的时候直接指定
CREATE TABLE mytable(  
ID INT NOT NULL,
username VARCHAR(16) NOT NULL,  
UNIQUE [indexName] (username(length))  
);  
```

#### 显示索引

```sql
SHOW INDEX FROM table_name; \G
```

**索引原理及优化请看[MySQL 索引及查询优化总结](https://juejin.im/post/5c2c8dace51d455d382ee046)**

### 字符集

#### 配置文件中增加默认字符集

```bash
vi /etc/my.cnf
在[client]下添加
default-character-set=utf8
在[mysqld]下添加
default-character-set=utf8
```

#### 修改字符集

```sql
修改数据库的字符集
mysql>use mydb
mysql>alter database delivery character set utf8;
创建数据库指定数据库的字符集
mysql>create database mydb character set utf-8;

set character_set_database=utf8;
set character_set_server=utf8;
```

#### 查看字符集

```bash
show variables like 'collation_%';
show variables like 'character_set_%';

#查看数据字符集
show create database shiyan\G
#查看表字符集
show table status from 库名 like 表名;
show table status from ctrp like  tb_resource_info;
```

#### utf8mb4

MySQL在5.5.3之后增加了这个utf8mb4的编码，mb4就是most bytes 4的意思，专门用来兼容四字节的unicode。好在utf8mb4是utf8的超集，除了将编码改为utf8mb4外不需要做其他转换。当然，为了节省空间，一般情况下使用utf8也就够了。

最早的计算机在设计时采用8个比特（bit）作为一个字节（byte），所以，一个字节能表示的最大的整数就是255（二进制11111111=十进制255），0 - 255被用来表示大小写英文字母、数字和一些符号，这个编码表被称为ASCII编码，比如大写字母A的编码是65，小写字母z的编码是122。

### 参考

[mysql索引](https://www.runoob.com/mysql/mysql-index.html)

[MySQL 索引及查询优化总结](https://juejin.im/post/5c2c8dace51d455d382ee046)

[MySQL查看和修改字符集的方法](https://www.cnblogs.com/yangmingxianshen/p/7999428.html)
