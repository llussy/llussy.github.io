---
layout: post
title: "Taints and Tolerations"
date: 2022-11-16 20:20:30 +0800
header-style: text
header-img: "img/anfeng.jpg"
catalog: ture
multilingual: false
tags:
    - kubernetes
---
### Taints 污点 and Tolerations 容忍

Taints and Tolerations 是pod的一个属性，它将允许某些pod在指定的节点上或者不允许指定的pod到指定节点上或者必须要有某些的pod才能调度到指定节点上。

可以通过`kubectl taint`来执行 Taints和Tolerations和搭配使用的，Taints定义在Node节点上，声明污点及标准行为，Tolerations定义在Pod，声明可接受得污点。

```bash
# kubectl taint nodes node1 key1=value1:NoSchedule
kubectl taint nodes node1 key=value:NoSchedule 
 -------node1:节点名称 
-------key 就是key 
-------value 就是value 
-------effect效果是 NoSchedule 
意思是说只有拥有: key=value:NoSchedule这个属性的pod才能调度到node1上


kubectl taint nodes node1 name=lucky:NoSchedule

spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  tolerations:
  - key: name
    value: lucky
    effect: NoSchedule

```

`operator`可以定义为：

-   Equal 表示key是否等于value，默认
-   Exists 表示key是否存在，此时无需定义value

`effect`可以定义为：

-   NoSchedule 表示不允许调度，已调度的不影响
-   PreferNoSchedule 表示尽量不调度
-   NoExecute 表示不允许调度，已调度的在tolerationSeconds（定义在Tolerations上）后删除

看下Master节点上的annotations定义：

```bash
annotations: scheduler.alpha.kubernetes.io/taints: '[{"key":"dedicated","value":"master","effect":"NoSchedule"}]'volumes.kubernetes.io/controller-managed-attach-detach: "true"
```

Master节点上定义了`Taints`，它表达的是一个含义：此节点已被key=value污染，Pod调度不允许（Pod Tolerates Node Taints策略）或尽量不（Taint Toleration Priority策略）调度到此节点，除非是能够容忍（Tolerations）key=value污点的Pod。

Master节点上定义了`Taints`，声明：如果不是带有`Tolerations`定义为`[{"key":"dedicated","value":"master","effect":"NoSchedule"}]`的Pod，不允许调度到Master节点，PS：`operator`的默认值为`Equal`，所以可以不必显示声明。

## reference
[Taints and Tolerations](https://blog.csdn.net/tiger435/article/details/73650174)

[官方文档](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/)

[K8S学习之污点容忍](https://blog.csdn.net/weixin_60092693/article/details/122643690)
