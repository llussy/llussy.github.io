---
layout: post
title: "kubernetes node可用内存"
date: 2022-11-16 21:20:30 +0800
header-style: text
header-img: "img/anfeng.jpg"
catalog: ture
multilingual: false
tags:
    - kubernetes
---
### kubernetes node可用内存

kubectl top node 发现内存使用率很高（超过100%），但是实际上内存还有很多可用，这是为什么呢？

```bash
# kubectl top node 
NAME    CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
node-1   553m         14%    11363Mi         91%       
node-2   400m         10%    10120Mi         81%       
node-3   224m         5%     10613Mi         85%       
node-4   1086m        27%    12535Mi         101% 
```
查看node可调度的内存

```bash
# kubectl describe node node-1 
Allocatable:
  cpu:                3900m
  ephemeral-storage:  190119300396
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             12670892Ki
  pods:               64

```

这是由于kubelet 可调度的内存不是所有的内存，需要去掉预留资源和`eviction-threshold`,这个预留资源是由kubelet的--kube-reserved和--system-reserved参数指定的,eviction临界点由kubelet的--eviction-hard参数指定的。

官方的一张图解释的很清楚
![](https://llussy.github.io/images/2022-11-17-17-38-09.png)

**让我们来计算一下**

kubelet的预留配置
```bash
# cat /etc/kubernetes/kubelet-customized-args.conf 
KUBELET_CUSTOMIZED_ARGS=" --eviction-hard=imagefs.available<15%,memory.available<300Mi,nodefs.available<10%,nodefs.inodesFree<5% --system-reserved=cpu=50m,memory=1272Mi --kube-reserved=cpu=50m,memory=1272Mi --kube-reserved=pid=1000 --system-reserved=pid=1000 "
```
free -m 查看系统总内存
```bash
# free -m
              total        used        free      shared  buff/cache   available
Mem:          15217       12300         236           5        2680        2682
Swap:             0           0           0
```

系统总内存减去kubelet预留的内存和eviction，就是node可用的内存
```bash
# kubelet Allocatable内存转换成Mi 
12670892/1024=12373Mi

# 系统总内存减去kubelet预留的内存
15217-1272-1272-300=12373Mi

```

## reference
[reserve-compute-resources](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/reserve-compute-resources/)

