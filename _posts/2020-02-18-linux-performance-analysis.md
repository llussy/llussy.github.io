---
layout: post
title: "linux性能分析"
date: 2020-02-18 12:40:30 +0800
catalog: ture
multilingual: false
tags:
    - linux
---
[toc]

### io相关

```bash
iotop
iostat -x 1 10
dstat  --top-io --top-bio
pidstat -d 1 10
```

### 网络

```bash
iftop
dstat -nf
sar -n DEV 1 10
nethogs
netstat -n | awk '/^tcp/ {++state[$NF]} END {for(key in state) print key,"\t",state[key]}'   #tcp链接情况
```

### cpu

```bash
pidstat -u 1 10
ps aux | sort -k3nr | head -n 10    #使用CPU前十

```

### 内存

```bash
pidstat -r 1 10
ps aux | sort -k4nr | head -n 10  # 使用内存前十

```

### 磁盘

```bash
# 写
time dd if=/dev/zero of=/data/test.dbf bs=8k count=300000 oflag=direct
# 加上 oflag=direct，测到的才是真实的磁盘IO速度

# 读
time dd if=/data/test.dbf of=/dev/null bs=8k count=300000



# hdparm -Tt /dev/sda     读取速度
/dev/sda:
 Timing cached reads:   14616 MB in  1.99 seconds = 7330.10 MB/sec
 Timing buffered disk reads: 1322 MB in  3.00 seconds = 440.56 MB/sec

# 查看sn
hdparm -I /dev/sdb | grep 'Serial Number'

```