---
layout: post
title: "kakfa常用命令"
date: 2020-07-28 09:32:30 +0800
catalog: ture
multilingual: false
tags:
    - elk
---

[toc]

#### 重启

```bash
/usr/local/kafka_2.11-2.0.0/bin/kafka-server-stop.sh
sleep 3
/usr/local/kafka_2.11-2.0.0/bin/kafka-server-start.sh  -daemon /usr/local/kafka_2.11-2.0.0/config/server.properties
```

#### 列出/删除topic

```bash
#列出topic
./bin/kafka-topics.sh --list --zookeeper localhost:2181

#列出topic详情
./bin/kafka-topics.sh --zookeeper lcoalhost:2181 --topic ecar-baojia --describe

#创建topic
./bin/kafka-topics.sh --create --topic headlineRecomUpdateSync4 --replication-factor 3 --partitions 5 --zookeeper  192.168.1.1:2181

#删除topic
./bin/kafka-topics.sh --delete --zookeeper localhost:2181 --topic test

#连接到zookeeper彻底删除topic
./bin/zookeeper-shell.sh 192.168.1.1:2181
找到topic目录：ls /brokers/topics   删掉对应的topic :     rmr  /brokers/topic/topic-name
找到目录:  ls /config/topics        删除对应的topic:     rmr  /config/topics/topic-name
```


####从 头开始消费数据

```shell
# old version
./bin/kafka-console-consumer.sh --zookeeper 192.168.1.1:2181 --topic ecar-baojia --from-beginning

# new version
./bin/kafka-console-consumer.sh --bootstrap-server 192.168.1.1:9092 --topic testtest --from-beginning

```

#### 查看/删除消费组

```bash
#查看消费组
./bin/kafka-consumer-groups.sh --bootstrap-server 192.168.1.1:9092 --list

#删除
./bin/kafka-consumer-groups.sh --bootstrap-server 192.168.1.1:9092 --group logstash-b --delete
```

####查看消费组的积压情况

```bash
# ./bin/kafka-consumer-groups.sh --bootstrap-server 192.168.1.1:9092 --describe --group logstash-a

TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                     HOST            CLIENT-ID
test-a         8          503108030       505454999       2346969         logstash-0-946898a5-7b1f-4930-9ec3-0d5d977d31b8 /192.168.1.1   logstash-0
test-a         2          503256691       505374825       2118134         logstash-0-946898a5-7b1f-4930-9ec3-0d5d977d31b8 /192.168.1.1   logstash-0
test-a         3          503152179       505364622       2212443         logstash-0-946898a5-7b1f-4930-9ec3-0d5d977d31b8 /192.168.1.1   logstash-0

```

展示的几个参数:
 TOPIC: 该消费者组消费的是哪些topic
 PARTITION: 表示该消费者消费的是哪些分区, 本例创建topic时只有一个分区, 所以输出只有一行, 分区号为0
 CURRENT-OFFSET: 表示消费者组最新消费的位移值, 此值在消费过程中是变化的
 LOG-END-OFFSET: 表示topic所有分区当前的日志终端位移值, 因为我们生产了1000万数据, 所以此处是1000万
 LAG: 表示滞后进度, 此值为LOG-END-OFFSET 与 CURRENT-OFFSET的差值, 代表的是滞后情况, 此值越大表示滞后严重, 本例最终LAG为0 说明没有消费滞后.

#### 检查消费者位置offset（0.9.0.0之前）

```bash
/usr/local/kafka_2.12-2.1.0/bin/kafka-run-class.sh kafka.tools.ConsumerOffsetChecker  --topic ugc_topic_test --zookeeper 192.168.1.1:2181
```

#### 清除offset

```bash
登陆kafka集群所使用的zookeeper集群中任一台服务器（以10.90.11.32为例）：

./ zkCli.sh -server 10.90.11.32:12181

[zk: 10.90.11.19:12181(CONNECTED) 1] ls /consumers/${consumer_name}/offsets/${topic_name}

[zk: 10.90.11.19:12181(CONNECTED) 2] rmr /consumers/${consumer_name}/offsets/${topic_name}

注：清除的是${consumer_name}消费者分组对于${topic_name}该topic的offset，此分组的消费者重启后默认会从当前开始消费。
```

#### 设置消费者offset

```bash
# 设置为最初偏移量
./kafka-consumer-groups.sh --bootstrap-server snn:6667 --group offsettest --topic offset-test --reset-offsets --to-earliest –execute

#设置任意偏移量
./kafka-consumer-groups.sh --bootstrap-server snn:6667 --group offsettest --topic offset-test --reset-offsets --to-offset 3 –execute

#设置最近偏移量
./kafka-consumer-groups.sh --bootstrap-server snn:6667 --group offsettest --topic offset-test --reset-offsets --to-latest --execute
```



#### 修改topic持久化时间

```bash
./bin/kafka-topics.sh --zookeeper 192.168.1.1:2181 -topic test --alter --config retention.ms=259200000
# 259200000 3天
# 345600000 4天
```



#### 增加partition分区数

```bash
./kafka-topics.sh --alter --topic recHeadLine --zookeeper  192.168.1.1:2181 --partitions 80
```

#### auto.offset.reset

```bash
earliest
当各分区下有已提交的offset时，从提交的offset开始消费；无提交的offset时，从头开始消费
latest
当各分区下有已提交的offset时，从提交的offset开始消费；无提交的offset时，消费新产生的该分区下的数据
none
topic各分区都存在已提交的offset时，从offset后开始消费；只要有一个分区不存在已提交的offset，则抛出异常
```

[Kafka auto.offset.reset值详解](https://blog.csdn.net/lishuangzhe7047/article/details/74530417)



#### 参考

[删除topics](https://www.cnblogs.com/felixzh/p/5992745.html)

[kafka删除topics](https://blog.csdn.net/xiaoyu_bd/article/details/52268647)

[kafka查询消费者组](https://www.jianshu.com/p/58276dd6e0e8)

[kafka手动修改消费者偏移量](<https://blog.csdn.net/only_1/article/details/83753109>)


