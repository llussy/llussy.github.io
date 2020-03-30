---
layout: post
title: "hadoop常用命令"
date: 2020-03-30 09:32:30 +0800
catalog: ture
multilingual: false
tags:
    - hadoop
---

### hdfs

```shell
sudo -u hdfs hdfs dfsadmin -report   #查看hdfs使用率
hdfs dfs -copyFromLocal /data/app_tmp/hippoZip/2018-05-29/ /hippo/hippo/hippoZip/ #上传
sudo -u hdfs hdfs dfs -mkdir /lisai  #创建目录

#检测损坏的block，然后删掉
hdfs fsck -list-corruptfileblocks
sudo -u hdfs hdfs dfs -rm  /tmp/hadoop-yarn/staging/history/done_intermediate/root/job_1486604935762_0002-1486605171955-root-Chukwa%2DDemux_20170209_09_52-1486605234028-0-0-FAILED-root.root-1486605178417.jhist
或者
hdfs fsck /xxx/xxx/xxx/xxx -delete

#根目录下各个目录的大小
hdfs dfs -du -h /

# 平衡限速
sudo -u hdfs hdfs dfsadmin -setBalancerBandwidth 5242880  # 5MB/s
#count
hdfs dfs -count < hdfs path >   或  hadoop fs -count < hdfs path >
统计hdfs对应路径下的目录个数，文件个数，文件总计大小
显示为目录个数，文件个数，文件总计大小，输入路径
```

### hadoop

```bash
hadoop fs -copyFromLocal 1.txt /user/lisai/
hadoop fs -ls /
hadoop fs -put /data/null_ad_imp/* /ids/no_ad_impl/
hadoop fs -get [hadoop源文件路径路径] [linux下载目的地路径]
hadoop fs -du -h /
hadoop classpath
```

### 集群间迁移数据

```bash
hadoop distcp hdfs://10.90.4.199:8020/newread/logbak/android/20170101 hdfs://10.21.8.65:8020/newread
hadoop distcp hdfs://10.90.4.199:8020/lijun hdfs://10.21.8.65:8020/lijun

```

#### 设置副本数

```bash
setrep

Usage: hadoop fs -setrep [-R] [-w] <numReplicas> <path>

Changes the replication factor of a file. If path is a directory then the command recursively
changes the replication factor of all files under the directory tree rooted at path.

Options:

The -w flag requests that the command wait for the replication to complete. This can potentially take a very long time.
The -R flag is accepted for backwards compatibility. It has no effect.

Example:

    hadoop fs -setrep -w 3 /user/hadoop/dir1

Exit Code:

Returns 0 on success and -1 on error.
---------------------
原文：https://blog.csdn.net/chengyuqiang/article/details/79139432
```
