---
layout: post
title: "mongo replicas"
date: 2020-02-24 15:40:30 +0800
catalog: ture
multilingual: false
tags:
    - db
---

[TOC]

### mongo节点

#### Priority 优先级

	优先级用于确定一个倾向成为主节点的程度。取值范围为0-100，Priority 0节点的选举优先级为0，不会被选举为Primary，这样的成员称为被动成员。
对于跨机房复制集的情形，如A，B机房，最好将『大多数』节点部署在首选机房，以确保能选择合适的Primary
对于Priority为0节点的情况，通常作为一个standby，或由于硬件配置较差，设置为0以使用不可能成为主

#### 隐藏节点（Hidden）

	Hidden节点不能被选为主（Priority为0）, 并且对Driver不可见。因Hidden节点不会接受Driver的请求，可使用Hidden节点做一些数据备份、离线计算的任务，不会影响复制集的服务，隐藏节点成员建议总是将其优先级设置为0`(priority 0)`。
由于对Driver不可见，因此不会作为read preference节点，隐藏节点可以作为投票节点

#### 投票节点（Vote）

投票节点不保存数据副本，不可能成为主节点 
Mongodb 3.0里，复制集成员最多50个，参与Primary选举投票的成员最多7个 
对于超出7个的其他成员（Vote0）的vote属性必须设置为0，即不参与投票



### mongo副本集搭建

#### 副本集原理

mongodb的复制至少需要两个节点。其中一个是主节点，负责处理客户端请求，其余的都是从节点，负责复制主节点上的数据。

mongodb各个节点常见的搭配方式为：一主一从、一主多从。

主节点记录在其上的所有操作oplog，从节点定期轮询主节点获取这些操作，然后对自己的数据副本执行这些操作，从而保证从节点的数据与主节点一致。

MongoDB复制结构图如下所示：

![MongoDB复制结构图](https://llussy.github.io/images/mongo/mongo-replicas.png)

以上结构图中，客户端从主节点读取数据，在客户端写入数据到主节点时， 主节点与从节点进行数据交互保障数据的一致性。



**先别开启认证，配置好副本集 创建好用户后 再开启**

#### 下载安装

```bash
yum install numactl -y
# 下载 mongodb-linux-x86_64-rhel70-3.6.7.tgz
tar -zxvf mongodb-linux-x86_64-rhel70-3.6.7.tgz -C /usr/local/

cd /usr/local/
rm -rf /usr/local/mongodb
mv mongodb-linux-x86_64-rhel70-3.6.7 mongodb

useradd mongodb
mkdir /data/mongodata
mkdir /data/logs/mongolog
chown -R mongodb:mongodb /usr/local/mongodb/
chown -R mongodb:mongodb /data/mongodata/
chown -R mongodb:mongodb /data/logs/mongolog/


echo 'export PATH=$PATH:/usr/local/mongodb/bin' >/etc/profile.d/mongo.sh
echo 'source /etc/profile.d/mongo.sh' >> /etc/profile
source /etc/profile

#key文件
openssl rand -base64 753 > keyfile
mv keyfile /data/mongodata/
chown -R mongodb.mongodb /data/mongodata/keyfile
chmod 600 /data/mongodata/keyfile
```

#### mongo配置文件

```bash
# mongo配置文件 cat /usr/local/mongodb/mongo/mongodb.conf
systemLog:
   destination: file
   path: "/data/logs/mongolog/mongodb.log"
   logAppend: true
   timeStampFormat: ctime 
storage:
   dbPath: /data/mongodata
   indexBuildRetry: true
   journal:
      enabled: true
      commitIntervalMs: 30
   directoryPerDB: true
   syncPeriodSecs: 60
   engine: wiredTiger
   wiredTiger:
      engineConfig:
         cacheSizeGB: 16
         journalCompressor: zlib
         directoryForIndexes: true
      collectionConfig:
         blockCompressor: zlib
      indexConfig:
         prefixCompression: true
processManagement:
   fork: true
   pidFilePath: /data/mongodata/mongo.pid
net:
#   bindIp: 127.0.0.1
   port: 27017
replication:
   replSetName: user-credit
###  启动后创建用户密码可以取消下面注释
#security:
#   keyFile: /data/mongodata/mongodb.key 
#  authorization: enabled
```

#### 启动mongo

```bash
sudo -u mongodb numactl --interleave=all /usr/local/mongodb/bin/mongod -f /usr/local/mongodb/mongodb.conf --auth --keyFile=/data/mongodata/keyfile
```



#### 创建副本集

```bash
config = {_id: 'user-credit', members: [
{_id: 0, host: '192.168.1.1:27017',priority:3},
{_id: 1, host: '192.168.1.2:27017',priority:2},
{_id: 2, host: '192.168.1.3:27017',priority:2},
{_id: 3, host: '192.168.1.4:27017',priority:2},
{_id: 4, host: '192.168.1.5:27017',arbiterOnly:true}
]};

rs.initiate(config);

#查看副本集状态
rs.status()
```

#### 副本集的状态

**stateStr**类型

```
1. STARTUP：刚加入到复制集中，配置还未加载  
2. STARTUP2：配置已加载完，初始化状态  
3. RECOVERING：正在恢复，不适用读  
4. ARBITER: 仲裁者  
5. DOWN：节点不可到达  
6. UNKNOWN：未获取其他节点状态而不知是什么状态，一般发生在只有两个成员的架构，脑裂  
7. REMOVED：移除复制集  
8. ROLLBACK：数据回滚，在回滚结束时，转移到RECOVERING或SECONDARY状态  
9. FATAL：出错。查看日志grep “replSet FATAL”找出错原因，重新做同步  
10. PRIMARY：主节点  
11. SECONDARY：备份节点 
```

**rs.status()**

```bash
wxh:PRIMARY> rs.status()
{

        "set" : "wxh",
        "date" : ISODate("2014-01-23T09:34:23Z"),
        "myState" : 1,
        "members" : [
                {
                        "_id" : 0,
                        "name" : "wxlab31:27017",
                        "health" : 1,
                        "state" : 2,
                        "stateStr" : "SECONDARY",   　　　　 　　#  stateStr用户描述服务器状态的字符串。有SECONDARY,PRIMARY,RECOVERING等
                        "uptime" : 30060,                    　　　　　　 #  uptime 从成员可到达一直到现在经历的时间，单位是秒。
                        "optime" : Timestamp(1390450194, 3),       
                        "optimeDate" : ISODate("2014-01-23T04:09:54Z"),      #  optimeDate 每个成员oplog最后一次操作发生的时间，这个时间是心跳报上来的，因此可能会存在延迟
                        "lastHeartbeat" : ISODate("2014-01-23T09:34:21Z"),    #  lastHeartbeat 当前服务器最后一次收到其他成员心跳的时间，如果网络故障等可能这个时间会大于2秒
                        "lastHeartbeatRecv" : ISODate("2014-01-23T09:34:21Z"),
                        "pingMs" : 0,　　　　　　　　　　　　　　  #  pinMs 心跳从当前服务器达到某个成员所花费的平均时间
                        "syncingTo" : "wxlab31:27018"　　　　　  　# syncingTo表示当前服务器从哪个节点在做同步
                },
                {
                        "_id" : 1,
                        "name" : "wxlab31:27018",
                        "health" : 1,
                        "state" : 1,
                        "stateStr" : "PRIMARY", 
                        "uptime" : 30067,
                        "optime" : Timestamp(1390450194, 3),
                        "optimeDate" : ISODate("2014-01-23T04:09:54Z"),
                        "self" : true  　　　　　　　　  # self 这个信息出现在执行rs.status( )函数的成员信息中
    　　     },
                {
                        "_id" : 2,
                        "name" : "wxlab31:27019",
                        "health" : 1,
                        "state" : 2,
                        "stateStr" : "SECONDARY",
                        "uptime" : 30066,            
                        "optime" : Timestamp(1390450194, 3),   
                        "optimeDate" : ISODate("2014-01-23T04:09:54Z"),
                        "lastHeartbeat" : ISODate("2014-01-23T09:34:22Z"),   
                        "lastHeartbeatRecv" : ISODate("2014-01-23T09:34:21Z"),
                        "pingMs" : 0, 
                        "syncingTo" : "wxlab31:27018"   
                },
                {
                        "_id" : 3,
                        "name" : "wxlab31:27020",
                        "health" : 1,
                        "state" : 2,
                        "stateStr" : "SECONDARY",
                        "uptime" : 30066,
                        "optime" : Timestamp(1390450194, 3),
                        "optimeDate" : ISODate("2014-01-23T04:09:54Z"),
                        "lastHeartbeat" : ISODate("2014-01-23T09:34:21Z"),
                        "lastHeartbeatRecv" : ISODate("2014-01-23T09:34:22Z"),
                        "pingMs" : 0,
                        "syncingTo" : "wxlab31:27018"
                }
        ],
        "ok" : 1
}

字段解释：
self 这个信息出现在执行rs.status()函数的成员信息中
stateStr用户描述服务器状态的字符串。有SECONDARY,PRIMARY,RECOVERING等
uptime 从成员可到达一直到现在经历的时间，单位是秒。
optimeDate 每个成员oplog最后一次操作发生的时间，这个时间是心跳报上来的，因此可能会存在延迟
lastHeartbeat 当前服务器最后一次收到其他成员心跳的时间，如果网络故障等可能这个时间会大于2秒
pinMs 心跳从当前服务器达到某个成员所花费的平均时间
errmsg 成员在心跳请求中返回的状态信息，通过是一些状态信息，不全是错误信息。
state和stateStr是重复的，都表示成员状态，只是state是内部的叫法。
health表示是否服务器可达，可达是1，不可达是0
optime与optimeDate表达的信息也是一样的，只是表示的方式不同，一个是用新纪元开始的毫秒数表示的，一个是用一种更容易阅读的方式表示。
syncingTo表示当前服务器从哪个节点在做同步。
由于rs.status()是从执行命令成员本身的角度得出的，由于网路等故障，这份报告可能不准确或者有些过时

```



#### mongo增加用户名密码

```bash
#以下操作在primary节点执行 切换到admin数据库
use admin

#添加root权限用户
db.createUser({user:"root",pwd:"passowrd",roles:[{role:"root",db:"admin"}]})

#添加数据库管理员
db.createUser({user:"dbadmin",pwd:"password",roles:[{role:"dbAdminAnyDatabase",db:"admin"}]})

#创建读写权限用户mongo
db.createUser({user:"mongo",pwd:"123456",roles:[{role:"readWrite",db:"product"}]})

#赋予dbOwner权限
db.grantRolesToUser('mongo',[{role:"dbOwner",db:"product"}]);
```

```bash
use user_credits

db.createUser({
	user:'user_credits',
	pwd:'password',
	customData:{description:"user_credits"},
	roles:[{
		'role':'dbOwner',
		'db':'user_credits'
	}]
})
```



##### mongo使用密码登录

`mongo -u'admin' -p'xxxx' --authenticationDatabase admin`

**连接远程数据库**

```shell
mongo 192.168.1.1:27017/admin -u root -p rootpwd
```

### mongo常用操作

#### mongo关闭启动

```
1. ps -ef | grep mongo | grep -v grep |awk '{print $2}'| xargs kill -9
2. use admin; db.shutdownServer();
3. /usr/local/mongodb/bin/mongod --shutdown --dbpath /data/mongodata/
```

启动脚本

```bash
if [ -d /sys/kernel/mm/transparent_hugepage ]; then
      thp_path=/sys/kernel/mm/transparent_hugepage
    elif [ -d /sys/kernel/mm/redhat_transparent_hugepage ]; then
      thp_path=/sys/kernel/mm/redhat_transparent_hugepage
    else
      return 0
fi

echo 0 > /proc/sys/vm/zone_reclaim_mode
echo 'never' > $thp_path/enabled
echo 'never' > $thp_path/defrag
echo 'no' > $thp_path/khugepaged/defrag

sudo -u mongodb numactl --interleave=all /usr/local/mongodb/bin/mongod -f /usr/local/mongodb/mongodb.conf --bind_ip_all
```



#### 打印同步信息

db.printSlaveReplicationInfo() [SlaveReplicationInfo](https://docs.mongodb.com/manual/reference/method/db.printSlaveReplicationInfo/)

#### 指定从库同步源

rs.syncFrom("10.90.70.170:27017") [syncFrom](https://docs.mongodb.com/manual/reference/method/rs.syncFrom/)

#### 让mongodb的secondary支持读操作

对于`replica set` 中的`secondary 节点`默认是不可读的。在写多读少的应用中，使用Replica Sets来实现读写分离。通过在连接时指定或者在主库指定`slaveOk`，由Secondary来分担读的压力，Primary只承担写操作。

如果通过shell访问mongo，要在secondary进行查询。会出现如下错误：

```bash
imageSet:SECONDARY>

 db.fs.files.find()

error: { "$err" : "not master and slaveOk=false", "code" : 13435 }
```



有两种方法实现从机的查询：

第一种方法：

`db.getMongo().setSlaveOk();`

第二种方法：`rs.slaveOk();`

但是这种方式有一个缺点就是，下次再通过mongo进入实例的时候，查询仍然会报错，为此可以通过下列方式

vi ~/.mongorc.js

增加一行rs.slaveOk();

这样的话以后每次通过mongo命令进入都可以查询了

#### cut_mongo_day.sh

```bash
date=`(date +"%Y%m")`
day=`(date +"%d")`
mongologdir=/data/logs/mongolog/
mongopid=/data/mongodata/mongo.pid

mkdir ${mongologdir}/${date}
execcut()
{
         cp ${mongologdir}/mongodb.log ${mongologdir}/${date}/${day}.log
         echo > ${mongologdir}/mongodb.log
         gzip ${mongologdir}/${date}/${day}.log

}

execcut
```





### 参考

[mongo索引](https://www.cnblogs.com/clnchanpin/p/6925915.html) 

[mongo索引原理](http://www.mongoing.com/archives/2797)

[让mongodb的secondary支持读操作](http://wengzhijuan12.blog.163.com/blog/static/3622414520137104257376/)

[rs.status](http://blog.itpub.net/22034023/viewspace-1074651/)

[mongo数据库常用命令](https://www.cnblogs.com/sunny3096/p/9181821.html)

[副本集选举](http://he-zhao.cn/2018/03/15/MongoDB-repSet/)








