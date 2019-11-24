---
layout: post
title: "elasticsearch常用操作"
date: 2019-06-22 14:55:34 +0800
catalog: ture  
multilingual: false
tags: 
    - elk
---



[TOC]

### 常用命令

查看node信息

```
curl -XGET 'http://ip:9200/_cat/nodes?pretty'
```

查看集群中的master

```bash
curl -XGET 'http://ip:9200/_cat/master?v'
```

创建删除索引

```bash
# 创建
curl -XPUT 'http://ip:9200/imcplog-2017-01-01'
curl -XPUT -u lisai:xxxx 'http://es.cloud.com:9200/anfeng-2018.11.20' -H 'Content-Type: application/json' -d '
{
  "settings": {  
        "index.number_of_shards" : 30,
        "index.number_of_replicas": 1
    }
}
'
# 删除
curl -XDELETE 'http://ip:9200/imcplog-2017-01-01'
```

查看索引列表

```bash
http://ip:9200/_cat/indices?v
```

查看节点信息

```bash
http://ip:9200/_nodes/stats/thread_pool?pretty
```

查看es线程情况

```
GET _nodes/stats/thread_pool?pretty
```

迁移分片
```bash
POST _cluster/reroute
{
 "commands" : [
{
"move" : {
"index" : "iflog-2018.11.20", "shard" : 3,
"from_node" : "es-11-58", "to_node" : "es-11-36"
}
}
]
}
```

### es模板
```bash
curl -u lisai:xxxxxx -XPUT 'http://es.cloud.com:9200/_template/cpdt-template?pretty' -H 'Content-Type: application/json' -d'
{
    "template" : "log--loda*",
    "order": 100,
    "settings" : {
        "index.number_of_shards" : 15,
        "number_of_replicas" : 1,
        "routing.allocation.total_shards_per_node": 3,
        "index.routing.allocation.exclude._ip" : "10.80.*.153",
        "refresh_interval" : "60s"
    }
}'
```

### 删除index
**shell**
```bash
#!/bin/bash

Indexs=`curl -XGET -u lisai:xxxx 'http://es.cloud.com:9200/_cat/indices' | awk '{print $3}' | egrep -v "^\."| grep -Ev "op-monitor-login|cache-eliminated"`
up_to_time=`date +%Y%m%d -d '10 days ago'`

for index in $Indexs;
do 
echo $index | egrep "[0-9]{4}\.[0-9]{2}\.[0-9]{2}" 
if [ $? == 0 ]
then 
  time=`echo $index | egrep -o "[0-9]{4}\.[0-9]{2}\.[0-9]{2}"| awk -F'.' '{print $1$2$3}'`
if [[ $up_to_time -gt $time ]]
then
curl -XDELETE  -u lisai:xxxx "http://es.cloud.com:9200/${index}"
fi
fi 
done
# 30 01 * * * /bin/sh /data/script/cron/delete_es_index.sh 1>/dev/null 2>&1
```

**python**
```python
#!/usr/bin/env python

import elasticsearch
import curator
import sys
import os


def main():
    if len(sys.argv) == 1:
        print('USAGE: [TIMEOUT=(default 120)] %s NUM_OF_DAYS HOSTNAME[:PORT] ...' % sys.argv[0])
        print('Specify a NUM_OF_DAYS that will delete indices that are older than the given NUM_OF_DAYS.')
        print('HOSTNAME ... specifies which ElasticSearch hosts to search and delete indices from.')
        sys.exit(1)

    client = elasticsearch.Elasticsearch(sys.argv[2:])

    ilo = curator.IndexList(client)
    empty_list(ilo, 'ElasticSearch has no indices')
    ilo.filter_by_regex(kind='prefix', value='jaeger-')
    ilo.filter_by_age(source='name', direction='older', timestring='%Y-%m-%d', unit='days', unit_count=int(sys.argv[1]))
    empty_list(ilo, 'No indices to delete')

    for index in ilo.working_list():
        print("Removing", index)
    timeout = int(os.getenv("TIMEOUT", 120))
    delete_indices = curator.DeleteIndices(ilo, master_timeout=timeout)
    delete_indices.do_action()


def empty_list(ilo, error_msg):
    try:
        ilo.empty_list_check()
    except curator.NoIndices:
        print(error_msg)
        sys.exit(0)

if __name__ == "__main__":
    main()


# 2 4 * * *  python /data/bin/esClean.py 5 ip:9200
```