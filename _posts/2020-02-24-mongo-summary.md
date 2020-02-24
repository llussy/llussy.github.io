---
layout: post
title: "mongo summary"
date: 2020-02-24 14:40:30 +0800
catalog: ture
multilingual: false
tags:
    - db
---

[toc]

### mongo介绍

MongoDB是一种nosql数据库。

#### 基本概念

- 文档：是MongoDB中数据的基本单元，非常类似于关系型数据库系统中的行（但是比行要复杂很多）
- 集合：就是一组文档，如果说MongoDB中的文档类似于关系型数据库中的行，那么集合就如同表

#### oplog

​oplog(操作日志)是local库下的一个**固定集合**，Secondary就是通过查看Primary 的oplog这个集合来进行复制的。每个节点都有oplog，记录这从主节点复制过来的信息，这样每个成员都可以作为同步源给其他节点。只能保存特定数量的操作日志,默认为磁盘的5%。**将oplog中的同一个操作执行多次，与执行一次的结果是一样的。**

#### mongo GridFS

GridFS 用于存储和恢复那些超过16M（BSON文件限制）的文件(如：图片、音频、视频等),也是文件存储的一种方式，但是它是存储在MonoDB的集合中; GridFS 可以更好的存储大于16M的文件。

### mongo常用命令

```bash
db.serverStatus()  # 查看服务器状态

#删除数据库
use test ; db.dropDatabase()

#创建删除集合
db.createCollection("runoob")
db.runoob.drop()

```

#### 启动关闭

```bash
#启动
sudo -u mongodb numactl --interleave=all /usr/local/mongodb/bin/mongod --bind_ip_all -f /usr/local/mongodb/mongodb.conf --auth --keyFile=/data/mongodata/mongodb.key

#关闭
1.  ps -ef | grep mongo | grep -v grep |awk '{print $2}'| xargs kill -9
2. use admin; db.shutdownServer();
3. /usr/local/mongodb/bin/mongod --shutdown --dbpath /data/mongodata/
4.killall mongod
```

#### mongo查询

```bash
use admin ; db.auth("user",'123456')  # 认证

#查询
db.event201812.find({"timestamp":{$gt: 1545843600,$lt:1545845400}}).count()
db.userInfo.find({name: /mongo/});  #包含mongo
db.userInfo.find({name: /^mongo/});  
升序：db.userInfo.find().sort({age: 1});  
降序：db.userInfo.find().sort({age: -1});  
```

##### 比较操作符

```bash
"$lt", "$lte", "$gt", "$gte", "$ne"就是全部的比较操作符，对应于"<", "<=", ">", ">=","!="。
```

##### $elemMatch

$elemMatch专门用于查询数组Field中元素是否满足指定的条件。

##### $in

MongoDB 数组array查询

##### $inc

The [`$inc`](https://docs.mongodb.com/manual/reference/operator/update/inc/#up._S_inc) operator increments a field by a specified value

##### $set

$set操作符替换掉指定字段的值

#### 副本集相关

```bash
rs.stepDown() #将Primary节点降级为Secondary节点 这个命令会让primary降级为Secondary节点，并维持60s，如果这段时间内没有新的primary被选举出来，这个节点可以要求重新进行选举。也可手动指定时间
rs.stepDown(30)

rs.freeze(100) #冻结Secondary节点  
rs.freeze() #解冻

db.adminCommand({"replSetMaintenance":true}) #强制Secondary节点进入维护模式

rs.isMaster(); #查看master

rs.add("")增加节点
rs.remove("") 删除节点

# 设置成员标签
conf = rs.config();
conf.members[0].tags={"dc" : "dc01"};
conf.members[1].tags={"dc" : "dc01"};
conf.members[2].tags={"dc" : "dc02"};
rs.reconfig(conf);

# 调整成员权重
conf=rs.config()
conf.members[1].priority=0
conf.members[1].votes=0
rs.reconfig(conf)
```

#### 分片相关

[MongoDB-Sharding常用命令](https://qwendy.github.io/2016/12/22/MongoDB-Sharding%E5%B8%B8%E7%94%A8%E5%91%BD%E4%BB%A4/)

### mongo角色管理

```bash
  Built-In Roles（内置角色）：
    1. 数据库用户角色：read、readWrite;
    2. 数据库管理角色：dbAdmin、dbOwner、userAdmin；
    3. 集群管理角色：clusterAdmin、clusterManager、clusterMonitor、hostManager；
    4. 备份恢复角色：backup、restore；
    5. 所有数据库角色：readAnyDatabase、readWriteAnyDatabase、userAdminAnyDatabase、dbAdminAnyDatabase
    6. 超级用户角色：root  
    // 这里还有几个角色间接或直接提供了系统超级用户的访问（dbOwner 、userAdmin、userAdminAnyDatabase）
    7. 内部角色：__system
```

**推荐使用readWrite角色 不使用dbAdmin角色**

#### 创建删除用户

**谨记：先在不开启认证的情况下，创建用户，之后关闭服务，然后再开启认证，才生效**

```bash
db.createUser({
	user:'root',
	pwd:'root',
	customData:{description:"管理员root"},
	roles:[{
		'role':'root',
		'db':'admin'
	}]
})

db.createUser({
	user:'user2',
	pwd:'user2',
	customData:{description:"数据库账户描述"},
	roles:[{
		'role':'readWrite',
		'db':'demo2'
	}]
})

db.createUser({
    user: 'test',
    pwd: 'test123',
    roles: [ { role: "dbAdmin", db: "testdb" } ]
})

#登录认证：
> db.auth("root","123456")

##查看现有用户：
use admin
db.system.users.find()   ##或 show users

###创建管理员
anfeng:PRIMARY> db.createUser({user:"lisai",pwd:"123456",roles:["root"]})
Successfully added user: { "user" : "lisai", "roles" : [ "root" ] }
anfeng:PRIMARY> db.auth('lisai','123456')

删除单个用户
db.system.users.remove({user:"XXXXXX"})
删除所有用户
db.system.users.remove({})
```

### mongo索引

​索引通常可以极大的提高查询的效率。假设没有索引。MongoDB在读取数据时必须扫描集合中的每一个文件并选取那些符合查询条件的记录。 
这样的扫描全集合的查询效率是很低的，特别在处理大量的数据时，查询能够要花费几十秒甚至几分钟，这对站点的性能是很致命的。
索引是特殊的数据结构，索引存储在一个易于遍历读取的数据集合中。索引是对数据库表中一列或多列的值进行排序的一种结构

#### 创建索引

MongoDB使用 ensureIndex() 方法来创建索引。

ensureIndex()方法基本的语法格式例如以下所看到的：

```bash
db.COLLECTION_NAME.ensureIndex({KEY:1})
```

语法中 Key 值为你要创建的索引字段，1为指定按升序创建索引，假设你想按降序来创建索引指定为-1就可以。

```bash
# 创建索引
db.userInfo.ensureIndex({name: 1});
db.userInfo.ensureIndex({name: 1, ts: -1}); #多个字段创建索引    1为升序 -1为降序
db.COLLECTION_NAME.reIndex()  # 重建集合所有索引 
# 后台创建索引
db.userInfo.ensureIndex({open: 1, close: 1}, {background: true})  #background:true 


# 查询索引
db.userInfo.getIndexes(）
db.system.indexes.find() # 查询数据库中所有索引
db.COLLECTION_NAME.totalIndexSize()  #查看索引大小



#删除索引
db.COLLECTION_NAME.dropIndex("INDEX-NAME")
#删除所有索引
db.COLLECTION_NAME.dropIndexes()
```

### mongo monitor

#### mongostat

>  mongostat是mongodb自带的状态检测工具，在命令行下使用，会间隔固定时间获取mongodb的当前运行状态，并输出。

```bash
mongostat --host 192.168.1.100:27017 -uroot -p123456 --authenticationDatabase admin
```

[各个参数详细信息,请点击](https://blog.csdn.net/u011186019/article/details/70918288)

### mongo backup

```bash
#!/bin/sh
DATE=`date +%Y%m%d`
DEL_DATE=$(date -d '-3 days' "+%Y%m%d")
HOST=localhost
PORT=10001
HOSTNAME=`hostname`
BACKUP_PATH="/data/backup/$DATE"
DATA_DIR="/data/mongodb"
echo "@@@@@@@@@@@@@@@@@@@@@@" >>/data/backup/mongodb_bak.log
date +%Y%m%d%H%M >>/data/backup/mongodb_bak.log

lock()
{
echo "db.fsyncLock()"|  mongo  --host $HOST --port $PORT  admin
}

execute()
{
  lock
  if [ $? -eq 0 ]
  then
    echo "`date` mongodb lock successfully!" >>/data/backup/mongodb_bak.log
  else
    echo "`date` mongodb lock fail!" >>/data/backup/mongodb_bak.log
  fi
}

execute

back()
{
rsync -avzP $DATA_DIR $BACKUP_PATH/
}

execute()
{
  back
  if [ $? -eq 0 ]
  then
    echo "mongodb back successfully!" >>/data/backup/mongodb_bak.log
  else
    echo "mongodb back fail!" >>/data/backup/mongodb_bak.log
  fi
}
execute

unlock()
{
echo "db.fsyncUnlock()"|  mongo  --host $HOST --port $PORT  admin
}

execute()
{
  unlock
  if [ $? -eq 0 ]
  then
    echo "`date` mongodb unlock successfully!" >>/data/backup/mongodb_bak.log
  else
    echo "`date` mongodb unlock fail!" >>/data/backup/mongodb_bak.log
  fi
}
echo "db.fsyncUnlock()"|  mongo  --host $HOST --port $PORT  admin >>/data/backup/mongodb_bak.log
rsync  -avzP $BACKUP_PATH 192.168.1.1::ucmongodb/$HOSTNAME/
echo "************`date` backup ok **************************" >>/data/backup/mongodb_bak.log
rm -rf "/data/backup/${DEL_DATE}/"
```

### 参考

[mongo oplog详解](https://www.cnblogs.com/Joans/p/7723554.html)

[MongoDB在线修改oplogSize](https://winway.github.io/2016/01/28/mongodb-change-oplogsize/)

[MongoDB GridFS](https://www.runoob.com/mongodb/mongodb-gridfs.html)

[mongodb监控工具mongostat](https://blog.csdn.net/u011186019/article/details/70918288)

[mongodb 查询条件](http://www.ttlsa.com/mongodb/mongodb-conditional-operators/)
