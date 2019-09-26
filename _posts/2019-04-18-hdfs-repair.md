---
layout: post
title: "hdfs 丢失块处理"
date: 2019-04-18 19:08:34 +0800
catalog: ture  
multilingual: false
tags: 
    - hadoop
    - cdh
---


**dfs使用状况**

```bash
hdfs dfsadmin -report
```

**查看是否有丢失块**

```bash
hdfs fsck /

#过滤损失块
hdfs fsck / | egrep -v '^\.+$' | grep -v replica | grep -v Replica

#删除丢失块
hdfs fsck / -delete (注意路径)
```

**查看文件详情**

```bash
hdfs fsck /path/to/corrupt/file -locations -blocks -files
```

可以定位机器查看相应日志。