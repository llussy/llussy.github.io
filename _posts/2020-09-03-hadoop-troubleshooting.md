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

![image-20200903094632428](https://llussy.github.io/images/image-20200903094632428.png)

**解决**

- 调整**Java Heap Size of Hive Metastore Server in Bytes** 到8G

- 去除hive节点datanode和yarn角色。

  



## datanode

### Caught unexpected exception in main loop. 

ValueError: too many values to unpack

集群扩容启动cloudera-scm-agent报错。

```bash
# 报错信息
[07/Sep/2020 11:50:39 +0000] 179811 MainThread agent        ERROR    Caught unexpected exception in main loop.
Traceback (most recent call last):
  File "/usr/lib64/cmf/agent/build/env/lib/python2.7/site-packages/cmf-5.9.1-py2.7.egg/cmf/agent.py", line 758, in start
    self._init_after_first_heartbeat_response(resp_data)
  File "/usr/lib64/cmf/agent/build/env/lib/python2.7/site-packages/cmf-5.9.1-py2.7.egg/cmf/agent.py", line 938, in _init_after_first_heartbeat_response
    self.client_configs.load()
  File "/usr/lib64/cmf/agent/build/env/lib/python2.7/site-packages/cmf-5.9.1-py2.7.egg/cmf/client_configs.py", line 682, in load
    new_deployed.update(self._lookup_alternatives(fname))
  File "/usr/lib64/cmf/agent/build/env/lib/python2.7/site-packages/cmf-5.9.1-py2.7.egg/cmf/client_configs.py", line 432, in _lookup_alternatives
    return self._parse_alternatives(alt_name, out)
  File "/usr/lib64/cmf/agent/build/env/lib/python2.7/site-packages/cmf-5.9.1-py2.7.egg/cmf/client_configs.py", line 444, in _parse_alternatives
    path, _, _, priority_str = line.rstrip().split(" ")
ValueError: too many values to unpack
```

网上查看资料都是让修改client_configs.py文件，之前也扩容过集群启动就没问题，感觉根本原因不是这个文件的问题。

查看java版本，发现是openjdk。 openjdk对cdh的支持不是很好，remove 掉openjdk,安装Java jdk启动正常。

```bash
# java -version  
openjdk version "1.8.0_181"
OpenJDK Runtime Environment (build 1.8.0_181-b13)
OpenJDK 64-Bit Server VM (build 25.181-b13, mixed mode)

#  java -version
java version "1.8.0_121"
Java(TM) SE Runtime Environment (build 1.8.0_121-b13)
Java HotSpot(TM) 64-Bit Server VM (build 25.121-b13, mixed mode)
```

