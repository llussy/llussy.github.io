---
layout: post
title: "kubernetes troubleshooting"
date: 2020-09-04 11:32:30 +0800
catalog: ture
multilingual: false
tags:
    - kubernetes
---

[toc]

## node相关

### node维护

由于硬件问题或其他原因，node节点需要停机维护。

```bash
1.禁止新调度 到node上
 kubectl cordon k8snode
 
2.驱逐这个node的容器 过程比较慢
kubectl drain k8snode --ignore-daemonsets

3.解锁禁止调度。恢复业务
kubectl uncordon k8snode
```



### 内核版本过低引起的bug

linux内核版本太低(centos7.6.1810 内核版本：3.10.0-957.el7.x86_64)，容器多次重启可能会导致整个OS Hang住无法执行任何命令,node 节点notready。 **建议升级内核版本到4.19+。**

详细问题：

1. 3.10 内核 kmem account 内存泄漏 bug导致节点 NotReady（SLUB: Unable to allocate memory on node -1）
2. 3.10 内核 Bug, 系统日志出现大量 kernel: unregister_netdevice: waiting for eth0 to become free 报错，节点 NotReady
3. kernel: NMI watchdog: BUG soft lockup -CPU#9 stuck for 22s! [migration/9:54]”

社区相关 Issue:

- https://github.com/kubernetes/kubernetes/issues/61937
- https://github.com/opencontainers/runc/issues/1725
- https://support.mesosphere.com/s/article/Critical-Issue-KMEM-MSPH-2018-0006
- https://github.com/opsnull/follow-me-install-kubernetes-cluster/issues/430

[诊断修复 TiDB Operator 在 K8s 测试中遇到的 Linux 内核问题](https://mp.weixin.qq.com/s?__biz=MzI3NDIxNTQyOQ==&mid=2247488656&idx=1&sn=e92c1ae5bd603e1f020fe51916d4daa5&chksm=eb1633fadc61baec932be490bc43c698945785ae4b8a799e72620092b021cda26e99e06ca288&mpshare=1&scene=1&srcid=&rd2werd=1#wechat_redirect)

[K8S 问题排查： cgroup 内存泄露问题](http://www.xuyasong.com/?p=2049)



### node 资源不够,导致pod被驱逐

遇到的问题：node 磁盘空间不足，导致pod被驱逐到其他node上，造成其他node也磁盘不足，会有连锁反应。

解决：紧急升级node磁盘，当时买的云主机系统盘比较小。

**node资源方面相关延伸**

```bash
# kubelet配置

1：设置预留系统服务的资源 
--system-reserved=cpu=200m,memory=1G

2：设置预留给k8s组件的资源（主要组件）
--kube-reserved=cpu=200m,memory=1G
系统内存--system-reserved-kube-reserved 就是可以分配给pod的内存

3：驱逐条件#驱逐条件只对内存硬盘有效，对CPU无效
--eviction-hard=nodefs.available<10%,nodefs.inodesFree<5%,memory.available<100Mi

4：最小驱逐
--eviction-minimum-reclaim="memory.available=0Mi,nodefs.available=500Mi,imagefs.available=2Gi"   #暂时不用

5：节点状态更新时间
--node-status-update-frequency =10s    #(默认10S，不要轻易修改。很容器node再read和notread问题) 

6：驱逐等待时间
--eviction-pressure-transition-period=60s
```

[kubeernetes节点资源限制](https://www.cnblogs.com/ssss429170331/p/7685163.html)

[从一次集群雪崩看Kubelet资源预留的正确姿势](<https://my.oschina.net/jxcdwangtao/blog/1629059>)

## pod相关

### 容器时区问题

确保容器时区正确，可挂载宿主机localtime。

**pod.spec.containers下：**

```yaml
    volumeMounts:
    - mountPath: /etc/localtime
      name: localtime
      readOnly: true
```

**pod.spec.volumes下：**

```yaml
  volumes:
  - hostPath:
      path: /etc/localtime
      type: ""
    name: localtime
```

java可能还需在启动时指定时区

`java -jar -Duser.timezone=GMT+08`



### QoS问题

**Quality of Service**

内存不足，oom calico优先被干掉，calico等系统组件没有设置request、limit。是BestEffort最低优先级，第一个被kill。

**QoS：**

Guaranteed

- Every Container in the Pod must have a memory limit and a memory request, and they must be the same.

- Every Container in the Pod must have a CPU limit and a CPU request, and they must be the same.

Burstable

- The Pod does not meet the criteria for QoS class Guaranteed.
- At least one Container in the Pod has a memory or CPU request.

Best-Effort

- For a Pod to be given a QoS class of BestEffort, the Containers in the Pod must not have any memory or CPU limits or requests.



### 低版本java不能自动识别容器限制

低版本java自动识别到容器限制，获取正确的内存和CPU信息.

需要添加额外参数控制内存 java -Xms1G -Xmx1G -XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap

8u191版本已经修复这个问题。https://www.oracle.com/java/technologies/javase/8u191-relnotes.html

[容器中的JVM资源该如何被安全的限制？](https://qingmu.io/2018/12/17/How-to-securely-limit-JVM-resources-in-a-container/?appinstall=0)

### 容器优雅关闭

**容器中的 1 号进程是非常重要的，如果它不能正确的处理相关的信号，那么应用程序退出的方式几乎总是被强制杀死而不是优雅的退出。**

K8S滚动更新时，会在启动新pod时慢慢终止旧pod，Kubernetes会向旧pod中的容器发送SIGTERM信号,并等待应用优雅的结束（k8s默认等待30s)。注意，只有容器中的 1 号进程能够收到信号，这一点非常关键!
究竟谁是 1 号进程则主要由 EntryPoint, CMD等指令的写法决定。

详细信息：[Dockerfile中CMD和ENTRYPOINT](https://llussy.github.io/2019/08/14/dockerfile-CMD-ENTRYPOINT/)

这里推荐使用exec模式的写法,这样1号进程就是你的程序，滚动升级时,它会接收到SIGTERM信号： 才会优雅的结束进程。

```bash
不要使用CMD command param1 param2 这种形式，这种形式是shell模式，1号进程会是sh。

# 错误案例
CMD /start.sh  #这种形式是shell模式，1号进程会是sh   滚动升级是杀掉的是sh，应用进程是sh下属进程，所以应用进程还会卡在容器里不会关闭,无法优雅的stop进程

# 正确用法
CMD ["executable","param1","param2"]
ENTRYPOINT ["executable", "param1", "param2"]

```

优雅关闭如果还会出现5xx问题，可以在k8s层面增加lifecycle prostop

If one of the Pod's containers has defined a preStop hook, the kubelet runs that hook inside of the container. If the preStop hook is still running after the grace period expires, the kubelet requests a small, one-off grace period extension of 2 seconds.

```yaml
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]
```

也可用 supervisord管理容器去拦截sigteam信号。

```bash
#!/bin/sh
pidfile=/var/tmp/supervisord.pid
signal_term(){
    echo "docker start exit...\n"
    pid=`cat $pidfile`
    sleep 5;
    kill -s 15 ${pid}
    while true
    do
        if [ ! -f $pidfile ];then
            echo "docker end exit...\n"
            exit;
        fi
        sleep 1;
    done;
}
trap signal_term 15;
# start up
supervisord -c /etc/supervisord.conf &
wait $!
```



### traefik节点与业务节点隔离

保障traefik性能 最好和其他业务节点隔离开。traefik有问题大部分业务都会有影响。

可以将traefik节点禁止调度，kubectl cordon xxx

## other

### 跨版本升级造成api不兼容

k8s跨版本版本升级时需要注意api的版本，一定要仔细阅读**CHANGELOG**.

https://github.com/kubernetes/kubernetes/tree/master/CHANGELOG

如1.16的改动：

All resources under `apps/v1beta1` and `apps/v1beta2` - use `apps/v1` instead  `daemonsets`, `deployments`, `replicasets` resources under `extensions/v1beta1` - use `apps/v1` instead




