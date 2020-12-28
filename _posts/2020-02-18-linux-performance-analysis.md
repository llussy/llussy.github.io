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

### 综合
```bash
top/top -c
htop
sar  
tsar # https://github.com/kongjian/tsar.git
dstat
```


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
netstat -a | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'  #tcp链接情况
mtr
```

### cpu

```bash
mpstat -P ALL 1  # 查看所有cpu核信息
vmstat 1   # 查看cpu使用情况以及平均负载
perf top -p pid -e cpu-clock # 跟踪进程内部函数级cpu使用情况
pidstat -u 1 10 # 可以加指定进程 -p pid
ps aux | sort -k3nr | head -n 10    #使用CPU前十
```

### 内存

```bash
pidstat -r 1 10 # 可以加指定进程 -p pid
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

### 进程

```bash
#查看所有进程
pstree -a
```

### prlimit 动态修改最大文件数
[利用prlimit动态修改应用进程的最大文件打开数](http://www.eryajf.net/5008.html)
```bash
prlimit --pid 5616 --nofile=65535
````

### 参考
[接入层问题故障定位](https://www.jianshu.com/p/0bbac570fa4c)