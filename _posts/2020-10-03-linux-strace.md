---
layout: post
title: "linux strace"
date: 2020-10-03 22:09:30 +0800
catalog: ture
multilingual: false
tags:
    - linux
---

[toc]

### strace

**参数**

-tt 在每行输出的前面，显示毫秒级别的时间
-T 显示每次系统调用所花费的时间
-v 对于某些相关调用，把完整的环境变量，文件stat结构等打出来。
-f 跟踪目标进程，以及目标进程创建的所有子进程
-e 控制要跟踪的事件和跟踪行为,比如指定要跟踪的系统调用名称
-o 把strace的输出单独写到指定的文件
-s 当系统调用的某个参数是字符串时，最多输出指定长度的内容，默认是32个字节
-p 指定要跟踪的进程pid, 要同时跟踪多个pid, 重复多次-p选项即可。

```bash
# 常用方法

strace -tt -T -f -e trace=file -o /data/log/strace.log -s 1024 ./nginx

strace  -tt -p 21298
```

从性能分析大师 Brendan Gregg 的测试结果得知，被 strace 追踪的目标进程的运行速度会降低 100 倍以上，这对生产环境来说将是个灾难。

### perf

```bash
# 调用 syscall 数量的 top 排行榜
perf top -F 49 -e raw_syscalls:sys_enter --sort comm,dso --show-nr-samples

# 显示超过一定延迟的系统调用信息
perf trace --duration 200

# 统计某个进程一段时间内系统调用的开销
perf trace -p $PID  -s

# 高延迟的调用栈信息
perf trace record --call-graph dwarf -p $PID -- sleep 10
```

#### Flame Graph

```bash
perf record -e cpu-clock -g -p 222
-g 选项是告诉perf record额外记录函数的调用关系
-e cpu-clock 指perf record监控的指标为cpu周期
-p 指定需要record的进程pid


# 使用火焰图展示结果 https://www.cnblogs.com/felixzh/p/8932984.html

1、Flame Graph项目位于GitHub上：https://github.com/brendangregg/FlameGraph
2、可以用git将其clone下来：git clone https://github.com/brendangregg/FlameGraph.git
 注意：git clone之后，下面用到的*.pl文件先给+x可执行权限，注意路径

我们以perf为例，看一下flamegraph的使用方法：
1、第一步
$perf record -e cpu-clock -g -p 28591
Ctrl+c结束执行后，在当前目录下会生成采样数据perf.data.
2、第二步
用perf script工具对perf.data进行解析
perf script -i perf.data &> perf.unfold
3、第三步
将perf.unfold中的符号进行折叠：
./stackcollapse-perf.pl perf.unfold &> perf.folded
注意：该命令可能有错误，错误提示在perf.folded
4、最后生成svg图：
./flamegraph.pl perf.folded > perf.svg
```

### 参考

[强大的strace命令用法详解](https://www.linuxidc.com/Linux/2018-01/150654.htm)

[线上环境 Linux 系统调用追踪](https://mp.weixin.qq.com/s/8W0u1fL9zpXVg8Km1JRNUg)
