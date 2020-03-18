---
layout: post
title: "couchbase介绍"
date: 2020-03-18 09:52:30 +0800
catalog: ture
multilingual: false
tags:
    - db
---

### couchbase

Couchbase服务器可以单独运行，也可以作为集群运行。在Couchbase集群里，运行一个或多个Couchbase实例。集群里所有节点是相等的，提供相同的功能和信息，没有层次结构或者拓扑的概念，也没有主节点、从节点之分。**整个集群共享每个独立节点的信息，每个节点负责对数据的一部分进行响应。**
集群是水平扩展的。要增加集群的容量，你只需加多一个节点。节点间没有父子关系或者层次结构。这意味着Couchbase在存储容量和性能方面，都可以做到线性扩容。

集群里的每个节点包含了集群管理器组件。集群管理器负责下述行为：

```bash
集群管理
节点管理
节点监控
可管理的REST API
统计报表
实时日志
Multitenancy
访问安全
```

### bucket

Couchbase通过使用buckets提供数据管理服务，buckets相当于关系数据库中的库，couchbase中没有表的概念，保存数据时，先建bucket，然后就直接插入数据了。buckets可以供集群中的多个客户端程序访问。couchbase通过Buckets组织，管理和分析数据资源。

couchbase中有两种类型的数据bucket，当启动couchbase服务的时候，可以选择需要的类型。

**memcached buckets**

只将数据存储在内存中。提供了一个分布式的(横向扩展)，纯内存的，key-value缓存。Memcached buckets 设计用于关系数据库的缓存，可以缓存经常访问的数据，由此减少web程序中数据库的查询次数。

**couchbase buckets**

存数据在内存和硬盘。提供高可用性和可动态重新配置的分布式数据存储，提供数据持久化和复制服务。couchbase buckets 100% 兼容开源的分布式缓存memcached。

[buckets-memory-and-storage](https://docs.couchbase.com/server/current/learn/buckets-memory-and-storage/buckets-memory-and-storage.html)

### Failover

数据的副本在整个集群里分布。对于couchbase类型的bucket你可以配置副本的数量，就是说在一个couchbase集群里对每份数据保存多少数量的副本。
在服务器发生故障时（不管是临时故障还是管理维护），可以使用称为failover的技术把故障节点标记为不可用，从而激活该服务器对应的副本vBuckets。
failover进程联系每个保存了副本的服务器，更新内部映射表，将客户端的请求映射到新的可用节点上。
可以手工执行failover，也可以使用内置的自动failover机制，在集群里的节点不可用超过一定时间后，failover自动打开。

**酌情考虑是否开启。**[automatic-failover](https://docs.couchbase.com/server/current/learn/clusters-and-availability/automatic-failover.htmli)

### 参考

[couchbase集群](https://www.cnblogs.com/sunwubin/p/3426801.html)

[Couchbase简单介绍](https://zhuanlan.zhihu.com/p/49962194)