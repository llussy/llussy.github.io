---
layout: post
categories: Kubernetes
title: "Pod的liveness和readiness探针"
date: 2018-12-18 00:05:34 +0800
description: Kubernetes Probe
header-style: text
catalog: ture
multilingual: false
tags: kubernetes
---

[toc]

### livenessProbe

kubelet使用`liveness probe`（存活探针）来确定何时重启容器。例如，当应用程序处于运行状态但无法做进一步操作，`liveness`探针将捕获到deadlock，重启处于该状态下的容器，使应用程序在存在bug的情况下依然能够继续运行下去。

### readinessProbe

​ kubelet使用`readiness probe`（就绪探针）来确定容器是否已经就绪可以接受流量。只有当Pod中的容器都处于就绪状态时kubelet才会认定该Pod处于就绪状态。该信号的作用是控制哪些Pod应该作为service的后端。如果Pod处于非就绪状态，那么它们将会被从service的load balancer中移除。

​ 在没有配置`readinessProbe`的资源对象中，pod中的容器启动完成后，就认为pod中的应用程序可以对外提供服务，该pod就会加入相对应的service，对外提供服务。但有时一些应用程序启动后，需要较长时间的加载才能对外服务，如果这时对外提供服务，执行结果必然无法达到预期效果，影响用于体验。

​ 比如使用tomcat的应用程序来说，并不是简单地说tomcat启动成功就可以对外提供服务的，还需要等待spring容器初始化，数据库连接没连上等等。对于spring boot应用，默认的actuator带有/health接口，可以用来进行启动成功的判断。

​ **Readiness和livenss probe可以并行用于同一容器。 使用两者可以确保流量无法到达未准备好的容器，并且容器在失败时重新启动**

有三种探测方式： `exec`     `http`    `tcp`     具体用法可参照官方文档： [文档地址](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)

**http例子**

```yaml
    livenessProbe:
      failureThreshold: 3
      httpGet:
        path: /httpchk.php
        port: 80
        scheme: HTTP
      initialDelaySeconds: 60
      periodSeconds: 60
      successThreshold: 1
      timeoutSeconds: 1
    name: newsclient-ucweb
    ports:
    - containerPort: 80
      protocol: TCP
    readinessProbe:
      failureThreshold: 3
      httpGet:
        path: /httpchk.php
        port: 80
        scheme: HTTP
      initialDelaySeconds: 60
      periodSeconds: 60
      successThreshold: 1
      timeoutSeconds: 1
```

参数说明：

```yaml
initialDelaySeconds：容器启动后第一次执行探测是需要等待多少秒。

periodSeconds：执行探测的频率。默认是10秒，最小1秒。

timeoutSeconds：探测超时时间。默认1秒，最小1秒。

successThreshold：探测失败后，最少连续探测成功多少次才被认定为成功。默认是1。对于liveness必须是1。最小值是1。

failureThreshold：探测成功后，最少连续探测失败多少次才被认定为失败。默认是3。最小值是1。
```



### 参考：

[配置Pod的liveness和readiness探针](https://www.kubernetes.org.cn/2362.html)

[Kubernetes应用健康检查](https://blog.csdn.net/asdf57847225/article/details/78269506)

[官方文档](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)

