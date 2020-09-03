---
layout: post
title: "hadoop troubleshooting"
date: 2020-09-03 09:32:30 +0800
catalog: ture
multilingual: false
tags:
    - hadoop
---

[toc]

## hive

### HIVEMETASTORE_PAUSE_DURATION

```bash
# 报错信息
The health test result for HIVEMETASTORE_PAUSE_DURATION has become bad: Average time spent paused was 37.1 second(s) (61.91%) per minute over the previous 5 minute(s). Critical threshold: 60.00%.
```

最近几天cdh集群hive总是在凌晨出现HIVEMETASTORE_PAUSE_DURATION错误。

**问题排查**

- 排查hive所在机器，发现负载特别高，进一步排查有个spark程序(程序有问题)，跑在hive所在机器负载特别高。 经沟通次spark程序已经没用了忘记下线了。就kill掉了spark程序。
- 查看hive日志发现 有`canary_test_db_hive_HIVEMETASTORE`相关报错，cdh日志中也发现`The health test result for HIVEMETASTORE_CANARY_HEALTH has become bad: The Hive Metastore canary failed to create a database.`
- 进一步查看CDH监控，发现hive metastore 的JVM Heap Memory Usage基本跑满了，发现此参数配置的居然是1.3G，当有大量任务是这点内存肯定不够通，监控图上发现内存趋于跑满，很可能是内存不够造成的异常。

![image-20200903094632428](https://llussy.github.io/_posts/image-20200903094632428.png)

**解决**

- 调整**Java Heap Size of Hive Metastore Server in Bytes** 到8G

- 去除hive节点datanode和yarn角色。

  




